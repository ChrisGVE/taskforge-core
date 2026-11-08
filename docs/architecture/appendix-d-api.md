# taskforge — Architecture, Appendix D: the library API sketch

This appendix belongs to `ARCHITECTURE.md` (6.1). It sketches the in-process library API of the `taskforge-core` crate,
module by module, and the MCP's side of the admission nexus. Signatures are realistic, not exhaustive; `Fut<T, E>`
abbreviates `impl Future<Output = Result<T, E>> + Send`. Port traits consumed generically use that return-position form,
because `async fn` in a public trait fails with `async_fn_in_trait` under denied warnings and lets no generic caller
require `Send`; `JxaRunner` stays boxed because the MCP holds `Arc<dyn JxaRunner>` (Appendix A, X9).

```rust
// base -------------------------------------------------------------------
pub struct Config { /* taskwarrior, omnifocus, state, backup, projects, sync, plan, loc,
                       markers: MarkerTable, syntax: SyntaxTable, log, mcp (8.1) */ }  // deny_unknown_fields
pub fn load_config(sources: &ConfigSources, home: &Path) -> Result<Config, ConfigError>;
pub enum ConfigSources {
    Cli { flags: GlobalFlags, file: Option<PathBuf> },   // flags > file > shipped defaults
    Mcp { args: McpArgs, file: Option<PathBuf> },        // clap: argument > variable; then file > defaults
    DefaultsOnly,                                        // a non-entry binary (LP1)
}
#[derive(clap::Args)] pub struct McpArgs { /* one #[arg(long, env = ..)] per channel key; the live switch,
                       mcp.access and mcp.owned_tags declared without env (8.1) */ }
pub struct GlobalFlags { pub config: Option<PathBuf>, pub db: Option<PathBuf>, pub state_dir: Option<PathBuf> }
pub enum EntryKind { Cli, Mcp, Live }
pub enum Mode { Live, Off }
pub enum Reason { NotEntryBinary, SandboxFlag, ForeignHome, SteeringEnv, ModeOff }  // LP1-LP5
pub struct ProfileFacts { pub entry_binary: bool, pub sandbox_flag: bool, pub home: PathBuf,
                          pub account_home: PathBuf, pub steering: Vec<OsString>,
                          pub default_file: Option<Mode>, pub switch: Option<Mode> }   // plain data
pub enum Decision { Live, Sandbox(Reason) }
pub fn decide(f: &ProfileFacts, kind: EntryKind) -> Decision;    // pure; 8.10 is the only rule; no token

// ownership (6.5 is the only definition)
pub struct AccessRoot;                                 // holds the root's name, `access`
pub const SEAL_OWNER_TAG: &str = "access/agent";     // objects created on seal; generated tasks (FD13, FD15)
pub enum RootState { Unique(OfId), Missing, Duplicated(Vec<OfId>) }
pub fn resolve_root(top_level_access_tags: &[OfId]) -> RootState;
pub enum TagClass { Root, Owner, BelowOwner, Ordinary }
pub fn classify_tag(chain: &[OfId], root: &RootState) -> TagClass;     // Missing/Duplicated: never Owner
pub struct OwnerTag { id: OfId, name: String }
pub struct OwnedTags(Vec<OwnerTag>);                   // one client's resolved list; may be empty
pub enum Ownership { Untagged, Owned(OfId), Ambiguous }
pub enum Relation { Own, Path, Foreign, Untagged }
pub fn ownership_of(tag_chains: &[Vec<OfId>], root: &RootState) -> Ownership;
pub fn relation(o: &Ownership, owned: &OwnedTags) -> Relation;        // Own | Foreign | Untagged
pub enum Access { Narrow, Global }
pub fn relation_in_scope(r: Relation, on_path: bool, access: Access) -> Relation;
pub struct FolderFacts { pub projects: Vec<Vec<Vec<OfId>>> }  // per direct project, any status: tag id chains
pub enum Owner { User, Tag(OfId) }
pub enum FolderOwners { None, One(Owner), Mixed }      // None and One(User) are the user's folder
pub fn folder_owners(facts: &FolderFacts, root: &RootState) -> FolderOwners;
pub fn would_mix(destination: &FolderOwners, incoming: &FolderOwners) -> bool;
pub trait Classify { fn kind(&self) -> ErrorKind; fn code(&self) -> &'static str; fn retryable(&self) -> bool; }
// newtypes with smart constructors: Uuid (twin_of: UUIDv5 of the object's id), ProjectPath, Tag / TwTag,
// Description (strict or from_untrusted), Note, RepoRelPath, Estimate, OfId

// state ------------------------------------------------------------------
pub struct StateLayout { /* StateLayout::new(home, &StateConfig); every path derives from it */ }
impl StateLayout { pub fn user_lock_dir(&self) -> PathBuf;     // <HOME>/.local/state/taskforge-user-locks
                   pub fn backups(&self) -> PathBuf; pub fn roots_file(&self) -> PathBuf; }
pub mod entry {
    pub mod names { pub const CLI: &str = "taskforge"; pub const MCP: &str = "omnifocus-mcp";
                    pub const LIVE: &str = "taskforge-live"; }
    pub struct Started { pub config: Config, pub flags: GlobalFlags, pub profile: Profile }
    pub enum Profile { Live(LiveToken), Sandbox(Reason) }
    pub struct LiveToken { _private: () }            // field private to this module; not Clone, not Default
    pub fn start(kind: EntryKind) -> Result<Started, ConfigError>;  // reads its own facts (8.10)
}
pub enum LockKey { SyncRun(DbKey), DbWrite(DbKey), OfGate }
impl LockSet { pub async fn db_write(&self, db: &DbKey, wait: Duration) -> Result<DbWriteGuard, Busy>;
               pub async fn sync_run_or_busy(&self, db: &DbKey) -> Result<SyncRunGuard, Busy>;  // try-lock
               pub async fn try_sync_run(&self, db: &DbKey) -> Option<SyncRunGuard>; }
impl RepoRoots { pub fn roots(&self, p: &ProjectPath) -> Vec<PathBuf>;
                 pub fn record(&self, p: &ProjectPath, root: &Path); }   // write a temp file, rename
impl BackupDir { pub fn write_export(&self, run: &str, tasks: &[TwTask]) -> Result<PathBuf, StateError>; }

// tw ---------------------------------------------------------------------
pub struct TwEnv { pub bin: PathBuf, pub taskrc: PathBuf, pub data: PathBuf, pub hooks: HooksPolicy,
                   pub timeout: Duration }
impl TwEnv { pub async fn resolve(cfg: &TaskwarriorConfig, home: &Path, db: Option<&Path>)
                 -> Result<TwEnv, TwError>; }     // bare --version, then `_get rc.data.location`
pub trait TaskStore: Send + Sync {
    fn export(&self, f: &Filter) -> Fut<Vec<TwTask>, TwError>;
    fn create(&self, new: &NewTask) -> Fut<TwTask, TwError>;
    fn apply(&self, id: Uuid, change: &Change, pre: Option<Precondition>) -> Fut<TwTask, TwError>;  // 6.3
    fn set_state(&self, id: Uuid, to: TwStateChange) -> Fut<TwTask, TwError>;   // done/delete/start/stop
}
// Filter: All | Uuid | Project{path, Exact|Subtree} | Status | Tag | Virtual | Attr{KnownAttr, op, value} |
//         And | Or | Not — no Raw, no free text. Change: description, project, tags, depends, annotate,
//         note, dates, estimate, kind, level, parent_uuid, loc, of_id, sync_base, sync_dismissed, planned,
//         computed; the three sync attributes only from SyncMeta, Core's write use cases refuse them (8).
//         Precondition { expected_modified } (6.3, 5.9); None (for the link and baseline write after a seal's
//         OmniFocus write, 4.2 step (3), 5.5): the first export's `modified` is the expected one; after a
//         settle wait the task is exported again and the keys are changed on that second export (6.3).

// of ---------------------------------------------------------------------
pub trait JxaRunner: Send + Sync { fn run_omnijs<'a>(&'a self, script: &'a str)     // unchanged
        -> Pin<Box<dyn Future<Output = Result<Value>> + Send + 'a>>; }
impl<R: JxaRunner + ?Sized> JxaRunner for Arc<R> { /* forwards */ }
pub trait Gated: JxaRunner {
    type Held<'g>: JxaRunner where Self: 'g;
    fn hold(&self, scripts: u32) -> Fut<Self::Held<'_>, OfError>; }    // the gate once for one MCP call
impl<R: Gated + ?Sized> Gated for Arc<R> { /* forwards */ }
pub struct ScriptProcess { /* program + limit; stdin delivery, timeout, kill, output classification */ }
impl ScriptProcess { pub fn stub(path: &Path, limit: Duration) -> Result<ScriptProcess, OfError>; }
                                                   // refuses the system osascript's canonical path
pub struct LiveRunner { /* the process for OSASCRIPT, its limit */ }
impl LiveRunner { pub fn open(t: LiveToken, limit: Duration) -> LiveRunner; }
pub struct GatedRunner<R: JxaRunner> { /* OfGate + inner */ }
pub struct DisabledRunner;                         // answers OmniFocusError::Disabled(reason)
pub enum OfRunner { Live(Arc<GatedRunner<LiveRunner>>), Disabled(DisabledRunner) }  // Gated by dispatch
pub enum Source { Database, VisibleSet(Vec<OfId>) }  // binds SOURCE: the database (default: global reads,
                                                     // writes, stdio) or the visible set (narrow reads)
pub struct GuardedRunner<H: JxaRunner> { /* the held runner; per target its id and expected owner-tag ids */ }
impl<H: JxaRunner> GuardedRunner<H> { pub fn new(held: H, targets: &[(OfId, Vec<OfId>)]) -> Self; }
// every element of a batch is a target; an inbox create has none. SOURCE is bound to the database.
// runs `{ const GUARD = …; compare each target's owner-tag ids; on a difference return stale_facts }` + script
pub enum Outcome<T> { Applied(T), Mismatch(OfItemState) }
pub enum ByPath { One(OfId), Several(Vec<OfId>), NotFound { under: Option<OfId>, rest: Vec<String> } }
// NotFound: the deepest segment found, by id, and the names still to create; Several: E_PLAN_VIOLATION at seal,
// the root skipped and reported at pull (4.3, 5.5)
pub trait OmniStore: Send + Sync {                 // sync-facing; OfScripts and DisabledOf
    fn ownership_facts(&self, paths: &[OwnerTagPath]) -> Fut<OwnershipFacts, OfError>;
    fn plan_items(&self, project: &OfId) -> Fut<Vec<OfItemState>, OfError>;  // `project`: the container the
        // by-id read returns for the root's linked phases, else `ByPath::One` (4.3)
    fn project_by_path(&self, path: &ProjectPath) -> Fut<ByPath, OfError>;  // 5.5
    fn items(&self, ids: &[OfId]) -> Fut<Vec<ItemLookup>, OfError>;          // found | not found
    fn owner_scan(&self, tags: &[OwnerTag]) -> Fut<Vec<OfItemState>, OfError>;
    fn unlinked_named(&self, container: &OfParent, name: &str) -> Fut<Vec<OfItemState>, OfError>; // 4.2
    fn create(&self, spec: &OfItemSpec, owner_tag: &OfId) -> Fut<OfItemState, OfError>;
        // OfItemSpec: an action, a group, or a project with the folders missing above it (ByPath::NotFound);
        // the same evaluation creates those folders untagged under `under`, by id, and the item with `owner_tag`
        // (4.2, 5.5)
    fn update(&self, id: &OfId, patch: &OfPatch, expected: &Expected) -> Fut<Outcome<OfItemState>, OfError>;
    fn move_to(&self, id: &OfId, parent: &OfParent, expected: &ExpectedPlacement)
        -> Fut<Outcome<OfItemState>, OfError>;
    fn set_state(&self, id: &OfId, to: OfStateChange, expected: &ExpectedState)   // Complete|Drop|Reopen
        -> Fut<Outcome<OfItemState>, OfError>;
    fn rename_project(&self, id: &OfId, name: &str, expected: &str) -> Fut<Outcome<OfItemState>, OfError>;
}
pub enum ProbeScope { Narrow(Vec<OwnerTagPath>), Global }   // Narrow: resolve among the visible set
pub struct ProbeRequest { pub scope: ProbeScope, pub idents: Vec<Ident>, pub descendants: bool,
                          pub destination: Option<Ident>, pub source_chain: bool }
pub fn admission_probe<R: JxaRunner>(r: &R, q: &ProbeRequest) -> Fut<ProbeResult, OfError>;   // 6.4
pub fn count_rows<R: JxaRunner>(r: &R, owned: &[OfId]) -> Fut<Vec<CountRow>, OfError>;
// CountRow { id, parent, project, folder, tag_chains, buckets }: the moved counting script's predicates per row
pub mod upward { pub fn reach(facts: &Facts, op: OpClass) -> Vec<OfId>; }        // completion and reopen

// sync (pure part) -------------------------------------------------------
pub struct SyncMeta { pub of_id: Option<OfId>, pub base: Baseline, pub dismissed: BTreeSet<ReportCode>,
                      pub unlinked: bool }   // one attribute `sync_dismissed`: the codes plus the flag;
                                             // `sync dismiss` takes report codes only (5.5)
pub struct Baseline { /* field -> (base_t, base_o) digests, the tag set as one value */ }
pub fn digest<T: Canonical>(value: &Option<T>) -> Digest;   // first 16 hex of SHA-256 of the canonical JSON
pub fn merge_scalar<T: Canonical + Clone>(base: Option<(Digest, Digest)>, tw: &Option<T>, of: &Option<T>,
    tw_modified: DateTime<Utc>, of_modified: DateTime<Utc>, tw_is_machine_output: bool) -> Merge<T>;
pub enum Merge<T> { Keep, TakeTw(Option<T>), TakeOf(Option<T>), Same,
                    Conflict { winner: Side, kept: Option<T>, lost: Option<T> } }
pub fn match_container_and_name(task: &TwTask, container: &OfParent, candidates: &[OfItemState],
    seal_owner_tag: &OfId) -> Match;   // 4.2; One | Several | NotFound. The pull calls it once per candidate task
    // and links a pair only when each is the other's only candidate (4.2, 4.3)
pub struct TagPolicy { /* protects every tag under the access root (5.4) */ }
pub fn plan_seal(scope: &SealScope, tw: &[TwTask], of: &[OfItemState], opts: &SealOptions) -> SealPlan;

// app --------------------------------------------------------------------
pub struct Core<T: TaskStore> { /* cfg, LockedStore<T>, RepoRoots, clock */ }    // delivery phases 1, 4
pub struct SyncCore<T: TaskStore, O: OmniStore> { /* Core<T> + O + BackupDir + LockSet */ }
pub enum OfBackend { Live(OfScripts<Arc<GatedRunner<LiveRunner>>>), Disabled(DisabledOf) }  // OmniStore
pub fn build_of(cfg: &Config, profile: Profile) -> (OfBackend, OfRunner);  // one token, one gated runner
impl<T: TaskStore> Core<T> {   // list, add, modify, project(cwd), plan_check, plan_schedule(project, opts)
    pub async fn loc_resolve(&self, uuid: Uuid, root: Option<&Path>, repair: bool) -> Result<ResolveOutcome>; }
impl<T: TaskStore, O: OmniStore> SyncCore<T, O> {
    pub async fn pull(&self) -> Result<PullReport>;              // takes the sync-run lock itself
    pub async fn seal(&self, scope: SealScope, opts: SealOptions) -> Result<SealReport>;  // Project(p) | Loose
    /* plan_rename, plan_repair, scaffold, link, unlink, relink, dismiss */ }
pub type DefaultCore = Core<TwCli>;
pub type DefaultSyncCore = SyncCore<TwCli, OfBackend>;

// taskforge-mcp, ownership/ (the MCP side of the admission nexus, 6.5) ------
pub trait Admission: Send + Sync {             // the server holds no runner; this is the only way to one
    type Runner: Gated;
    const SUPPORTED_VERSIONS: &'static [ProtocolVersion];  // ClientPolicy: 2026-07-28; Unrestricted: KNOWN
    fn admit<'a>(&'a self, call: &'a Call<'a>) -> Fut<Admitted<'a, Self::Runner>, AdmitError>;
}
pub struct Admitted<'a, R: Gated> { /* AdmittedRunner (owns the hold), resolved ids, filter plan, limit */ }
impl<'a, R: Gated> Admitted<'a, R> {
    pub fn runner(&self) -> &AdmittedRunner<'a, R>;
    pub fn ids(&self) -> &ResolvedIds;
    pub fn finish(self, result: Value) -> Result<Value, AdmitError>;  // row filter, re-sort, limit, strip tagIds
}
pub enum AdmittedRunner<'a, R: Gated + 'a> { Guarded(GuardedRunner<R::Held<'a>>),   // HTTP writes
    Held(R::Held<'a>, Source),                   // HTTP reads: the visible set (narrow) or the database
    Direct(&'a R) }                              // stdio: the gated runner, the gate per script as today
pub struct ClientPolicy<R: Gated> { /* runner, owned-tag paths, access */ }   // HTTP only
pub enum Cell { Yes, Absent, Refused }          // a matrix cell carries no condition
pub struct Unrestricted<R: Gated> { /* runner */ }   // stdio only; private constructor
pub struct OmniFocusServer<A: Admission> { /* admission: A; no runner field */ }
pub type HttpServer<R> = OmniFocusServer<ClientPolicy<R>>;
pub type StdioServer<R> = OmniFocusServer<Unrestricted<R>>;
```
