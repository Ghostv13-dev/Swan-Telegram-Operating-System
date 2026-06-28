
STOS V2.3.2

SWAN Telegram Operating System


One owner. One bot. Buttons replace typing. Every feature must save the owner time.



Architecture

INTERNAL MODULES (Layer 1) — Generate intent only. Never mutate state.
  ├── Content Engine
  ├── Button Engine
  ├── Automation Engine
  ├── Community Engine
  ├── Customer Service Engine
  └── Delivery Engine

RUNTIME SERVICES (Layer 2) — Validate intent. Mutate state. Commit reality.
  └── 13-step pipeline:
      1. Event Ingestion
      2. Identity Resolution
      3. Permission Validation
      4. Route Resolution
      5. Execution Planning
      6. FSM Processing
      7. KV Atomic Commit ← all 5 outputs committed in one transaction
      8. Outbox Management
      9. Queue Processing
      10. Delivery Coordination
      11. Audit Recording
      12. Idempotency Control
      13. Lock Management

EXTERNAL TOOLS (Layer 3) — Execute effects. Never mutate STOS state.
  ├── Telegram Bot API
  ├── Deno Runtime / Deno KV
  └── HTTPS Webhook


Deployment

1. Create a Telegram bot


Message @BotFather → /newbot

Copy the token


2. Get your Telegram user ID


Message @userinfobot

Copy your numeric ID


3. Deploy to Deno Deploy


Push this repo to GitHub

Go to dash.deno.com → New Project → Import from GitHub

Set entry point: main.ts

Set environment variables (from .env.example):
BOT_TOKEN
WEBHOOK_SECRET (generate: openssl rand -hex 32)
OWNER_ID
WEBHOOK_URL (your Deno Deploy URL, e.g. https://stos.deno.dev)


4. Register the webhook

BOT_TOKEN=... WEBHOOK_SECRET=... WEBHOOK_URL=https://stos.deno.dev \
  deno run --allow-env --allow-net scripts/setup-webhook.ts

5. Message your bot /start

The owner control panel appears.



File Structure

stos/
├── main.ts ← Server entry point, webhook handler
├── deno.json ← Project config + task runner
├── .env.example ← Required environment variables
├── scripts/
│ └── setup-webhook.ts ← One-time webhook registration
└── src/
    ├── types/
    │ └── index.ts ← All types and interfaces
    ├── engines/ ← Layer 1: Internal Modules
    │ ├── button-engine.ts ← All inline keyboard rendering
    │ ├── content-engine.ts ← Post lifecycle management
    │ ├── automation-engine.ts ← Broadcasts, reminders, scheduler
    │ ├── community-engine.ts ← Member ingestion, join requests
    │ ├── customer-service-engine.ts ← FAQ, guides, tickets
    │ └── delivery-engine.ts ← Outbox entry builders
    ├── runtime/ ← Layer 2: Runtime Services
    │ ├── pipeline.ts ← 13-step processing pipeline
    │ ├── router.ts ← Step 4: Route resolution
    │ ├── kv.ts ← Deno KV access + atomic commit
    │ ├── delivery.ts ← Step 10: Delivery coordination
    │ └── queue-worker.ts ← Background job processor
    └── utils/
        └── telegram.ts ← Layer 3: Telegram API wrapper


Invariants (never violate these)

# Rule
INV-01 All mutations commit in one atomic transaction
INV-02 Internal Modules produce intent only. Never side effects
INV-03 Only Runtime Services may mutate persistent state
INV-04 Outbox entries written inside the atomic commit
INV-05 Idempotency check occurs before execution begins
INV-06 Lock acquired before FSM processing begins
INV-07 Audit entries are permanent and immutable
INV-08 STOS has exactly one owner. Always singular
INV-09 Webhook always returns 200 to Telegram
INV-10 Secret token validated on every incoming request
INV-11 Modules do not share state or call each other
INV-12 Pipeline steps execute in fixed sequential order
INV-13 Telegram is an output surface only. Never drives FSM directly


Security


Webhook requests without correct secret token → 403 (no processing)

BOT_TOKEN and WEBHOOK_SECRET live only in Deno Deploy env vars

Never in source code. Never in logs. Never in responses.

Three roles: OWNER / MEMBER / GUEST

KV namespaces isolated by key prefix, not if-checks



If code diverges from the specification, the code is wrong.




// ========================================================================
// STOS V2.3.2 — MAIN SERVER
// Deployment: GitHub → Deno Deploy → HTTPS Webhook → Runtime Pipeline
// Security boundary: webhook secret validated on EVERY request. (INV-10)
// Always returns 200 to Telegram. (INV-09)
// ===
=====================================================================

import { runPipeline } from "./src/runtime/pipeline.ts";
import { startQueueWorker } from "./src/runtime/queue-worker.ts";
import { setWebhook, getMe } from "./src/utils/telegram.ts";
import type { TelegramUpdate } from "./src/types/index.ts";

// -----------------------------------------------------------------------
// ENVIRONMENT VALIDATION
// -----------------------------------------------------------------------

const BOT_TOKEN = Deno.env.get("BOT_TOKEN");
const WEBHOOK_SECRET = Deno.env.get("WEBHOOK_SECRET");
const OWNER_ID = Deno.env.get("OWNER_ID");
const WEBHOOK_URL = Deno.env.get("WEBHOOK_URL");

if (!BOT_TOKEN) throw new Error("BOT_TOKEN environment variable is required.");
if (!WEBHOOK_SECRET) throw new Error("WEBHOOK_SECRET environment variable is required.");
if (!OWNER_ID) throw new Error("OWNER_ID environment variable is required.");

// -----------------------------------------------------------------------
// BOOT
// -----------------------------------------------------------------------

async function boot(): Promise<void> {
  // Start queue worker for background processing
  await startQueueWorker();

  // Register webhook if URL is provided
  if (WEBHOOK_URL) {
    try {
      const me = await getMe();
      console.log(`[Boot] Connected as @${me.username} (id=${me.id})`);
      await setWebhook(WEBHOOK_URL);
      console.log(`[Boot] Webhook registered at ${WEBHOOK_URL}`);
    } catch (err) {
      console.error("[Boot] Failed to register webhook:", err);
    }
  }

  console.log("[Boot] STOS V2.3.2 ready.");
}

// -----------------------------------------------------------------------
// REQUEST HANDLER
// -----------------------------------------------------------------------

async function handleRequest(req: Request): Promise<Response> {
  const url = new URL(req.url);

  // ── HEALTH CHECK ─────────────────────────────────────────────────
  if (url.pathname === "/health") {
    return new Response(JSON.stringify({ status: "ok", version: "2.3.2" }), {
      headers: { "Content-Type": "application/json" },
    });
  }

  // ── WEBHOOK ENDPOINT ─────────────────────────────────────────────
  if (url.pathname === "/webhook" && req.method === "POST") {
    return await handleWebhook(req);
  }

  return new Response("Not Found", { status: 404 });
}

// -----------------------------------------------------------------------
// WEBHOOK HANDLER
// Security validated here. (INV-10, Section 6)
// -----------------------------------------------------------------------

async function handleWebhook(req: Request): Promise<Response> {
  // ── SECURITY BOUNDARY ─────────────────────────────────────────────
  // Validate secret token before any processing. (Section 6)
  const secretHeader = req.headers.get("X-Telegram-Bot-Api-Secret-Token");
  if (secretHeader !== WEBHOOK_SECRET) {
    console.warn("[Security] Rejected request — invalid secret token.");
    return new Response("Forbidden", { status: 403 });
  }

  let update: TelegramUpdate;
  try {
    update = await req.json();
  } catch {
    console.error("[Webhook] Failed to parse request body.");
    return new Response("OK", { status: 200 }); // INV-09: always 200
  }

  // Run the 13-step pipeline asynchronously.
  // Response returns immediately so Telegram doesn't retry. (INV-09)
  runPipeline(update).catch((err) => {
    console.error("[Webhook] Unhandled pipeline error:", err);
  });

  return new Response("OK", { status: 200 });
}

// -----------------------------------------------------------------------
// SERVER START
// -----------------------------------------------------------------------

await boot();

Deno.serve({ port: 8000 }, handleRequest);





// ========================================================================
// STOS V2.3.2 — RUNTIME PIPELINE
// Layer 2: Runtime Services. The only layer that mutates state.
// Implements the fixed 13-step pipeline from Section 2.
// ========================================================================

import type {
  TelegramUpdate,
  ResolvedContext,
  ExecutionPlan,
  AtomicCommitPayload,
  AuditEntry,
  OutboxEntry,
  QueueItem,
  PipelineResult,
  UserRecord,
  FSMState,
  ActionType,
} from "../types/index.ts";

import {
  isAlreadyProcessed,
  markProcessed,
  acquireLock,
  releaseLock,
  getUser,
  upsertUser,
  atomicCommit,
  Keys,
} from "./kv.ts";

import { planToOutboxEntries } from "../engines/delivery-engine.ts";
import { router } from "./router.ts";
import { executeDelivery } from "./delivery.ts";

// -----------------------------------------------------------------------
// OWNER CONFIGURATION
// Read from environment at boot.
// -----------------------------------------------------------------------

const OWNER_ID = Number(Deno.env.get("OWNER_ID") ?? "0");

// -----------------------------------------------------------------------
// PIPELINE ENTRY POINT
// -----------------------------------------------------------------------

export async function runPipeline(update: TelegramUpdate): Promise<PipelineResult> {
  // ── Step 1: EVENT INGESTION ─────────────────────────────────────────
  const updateId = update.update_id;

  // ── Step 12: IDEMPOTENCY CHECK ──────────────────────────────────────
  // Checked early so we skip all work for replayed updates. (INV-05)
  const alreadyProcessed = await isAlreadyProcessed(updateId);
  if (alreadyProcessed) {
    return { success: true, httpStatus: 200 };
  }

  // ── Step 2: IDENTITY RESOLUTION ─────────────────────────────────────
  const ctx = resolveContext(update);
  if (!ctx) {
    // Unrecognized update shape — acknowledge and skip
    return { success: true, httpStatus: 200 };
  }

  // ── Step 13: LOCK MANAGEMENT ─────────────────────────────────────────
  // Per-user lock prevents concurrent mutations. (INV-06)
  const lockResource = `user:${ctx.actorId}`;
  const lockId = await acquireLock(lockResource);
  if (!lockId) {
    // Lock held by concurrent request — Telegram will retry
    return { success: true, httpStatus: 200 };
  }

  try {
    // ── Step 3: PERMISSION VALIDATION ──────────────────────────────────
    const actor = await resolveActor(ctx);

    // ── Step 4: ROUTE RESOLUTION ───────────────────────────────────────
    const plan = await router(ctx, actor);

    // ── Step 5: EXECUTION PLANNING ────────────────────────────────────
    // Plan is already immutable; captured as const.
    const immutablePlan: ExecutionPlan = Object.freeze({ ...plan });

    // ── Step 6: FSM PROCESSING ────────────────────────────────────────
    const { updatedActor, newFsmState } = computeFSMTransition(actor, immutablePlan);

    // ── Step 7: KV ATOMIC COMMIT ──────────────────────────────────────
    const outboxEntries = planToOutboxEntries(immutablePlan);
    const queueItems = buildQueueItems(outboxEntries);
    const auditEntry = buildAuditEntry(ctx, immutablePlan);
    const userEntry = await upsertUser(updatedActor);

    const commitPayload: AtomicCommitPayload = {
      stateUpdates: [userEntry],
      auditEntry,
      aggregateUpdates: [{ key: Keys.aggregate("total_updates"), delta: 1 }],
      outboxEntries,
      queueItems,
    };

    const committed = await atomicCommit(commitPayload);
    if (!committed) {
      throw new Error("Atomic commit failed — possible version conflict");
    }

    // ── Step 8: OUTBOX MANAGEMENT ─────────────────────────────────────
    // Outbox entries written inside atomic commit above. (INV-04)

    // ── Step 9: QUEUE PROCESSING ──────────────────────────────────────
    // Queue items enqueued inside atomic commit above.

    // ── Step 10: DELIVERY COORDINATION ───────────────────────────────
    await executeDelivery(outboxEntries);

    // ── Step 11: AUDIT RECORDING ──────────────────────────────────────
    // Audit written inside atomic commit above. (INV-07)

    // Mark as processed after all work completes.
    await markProcessed(updateId);

    return { success: true, httpStatus: 200 };
  } catch (err) {
    console.error(`[Pipeline] Error processing update ${updateId}:`, err);
    return { success: false, httpStatus: 200, error: String(err) };
    // Always return 200 to Telegram. (INV-09)
  } finally {
    await releaseLock(lockResource, lockId!);
  }
}

// -----------------------------------------------------------------------
// STEP 2: IDENTITY RESOLUTION
// -----------------------------------------------------------------------

function resolveContext(update: TelegramUpdate): ResolvedContext | null {
  if (update.message) {
    const msg = update.message;
    const from = msg.from;
    if (!from) return null;

    const chatType = msg.chat.type === "private" ? "user"
      : msg.chat.type === "channel" ? "channel"
      : "group";

    return {
      updateId: update.update_id,
      actorId: from.id,
      actorRole: from.id === OWNER_ID ? "OWNER" : "MEMBER",
      chatId: msg.chat.id,
      chatType,
      isOwner: from.id === OWNER_ID,
      messageId: msg.message_id,
      text: msg.text,
      rawUpdate: update,
    };
  }

  if (update.callback_query) {
    const cb = update.callback_query;
    const from = cb.from;
    const chatId = cb.message?.chat.id ?? from.id;
    const chatType = cb.message?.chat.type === "private" ? "user"
      : cb.message?.chat.type === "channel" ? "channel"
      : "group";

    return {
      updateId: update.update_id,
      actorId: from.id,
      actorRole: from.id === OWNER_ID ? "OWNER" : "MEMBER",
      chatId,
      chatType,
      isOwner: from.id === OWNER_ID,
      messageId: cb.message?.message_id,
      callbackQueryId: cb.id,
      callbackData: cb.data,
      rawUpdate: update,
    };
  }

  if (update.my_chat_member) {
    const from = update.my_chat_member.from;
    return {
      updateId: update.update_id,
      actorId: from.id,
      actorRole: from.id === OWNER_ID ? "OWNER" : "MEMBER",
      chatId: update.my_chat_member.chat.id,
      chatType: "group",
      isOwner: from.id === OWNER_ID,
      rawUpdate: update,
    };
  }

  if (update.chat_join_request) {
    const from = update.chat_join_request.from;
    return {
      updateId: update.update_id,
      actorId: from.id,
      actorRole: "GUEST",
      chatId: update.chat_join_request.chat.id,
      chatType: "group",
      isOwner: false,
      rawUpdate: update,
    };
  }

  return null;
}

// -----------------------------------------------------------------------
// STEP 3: PERMISSION VALIDATION + ACTOR RESOLUTION
// -----------------------------------------------------------------------

async function resolveActor(ctx: ResolvedContext): Promise<UserRecord> {
  const existing = await getUser(ctx.actorId);

  if (existing) return existing;

  // New user — create record
  const user = ctx.rawUpdate.message?.from ?? ctx.rawUpdate.callback_query?.from;
  const now = Date.now();

  return {
    id: ctx.actorId,
    firstName: user?.first_name ?? "Unknown",
    lastName: user?.last_name,
    username: user?.username,
    role: ctx.isOwner ? "OWNER" : "MEMBER",
    fsmState: "IDLE",
    fsmContext: {},
    joinedAt: now,
    updatedAt: now,
  };
}

// -----------------------------------------------------------------------
// STEP 6: FSM PROCESSING
// -----------------------------------------------------------------------

const FSM_TRANSITIONS: Partial<Record<ActionType, FSMState>> = {
  CREATE_TICKET: "TICKET_OPEN",
  CLOSE_TICKET: "TICKET_CLOSED",
  RESOLVE_TICKET: "TICKET_RESOLVED",
};

function computeFSMTransition(
  actor: UserRecord,
  plan: ExecutionPlan
): { updatedActor: UserRecord; newFsmState: FSMState } {
  const newFsmState = FSM_TRANSITIONS[plan.action] ?? actor.fsmState;

  const updatedActor: UserRecord = {
    ...actor,
    fsmState: newFsmState,
    fsmContext: {
      ...actor.fsmContext,
      ...(plan.metadata.postId ? { activePostId: plan.metadata.postId } : {}),
      ...(plan.metadata.ticketId ? { activeTicketId: plan.metadata.ticketId } : {}),
    },
    updatedAt: Date.now(),
  };

  return { updatedActor, newFsmState };
}

// -----------------------------------------------------------------------
// AUDIT ENTRY BUILDER
// -----------------------------------------------------------------------

function buildAuditEntry(ctx: ResolvedContext, plan: ExecutionPlan): AuditEntry {
  return {
    id: crypto.randomUUID(),
    timestamp: Date.now(),
    actorId: ctx.actorId,
    action: plan.action,
    details: {
      chatId: ctx.chatId,
      callbackData: ctx.callbackData,
      text: ctx.text?.slice(0, 100),
      metadata: plan.metadata,
    },
    chatId: ctx.chatId,
  };
}

// -----------------------------------------------------------------------
// QUEUE ITEM BUILDERS
// -----------------------------------------------------------------------

function buildQueueItems(outboxEntries: OutboxEntry[]): QueueItem[] {
  if (outboxEntries.length === 0) return [];

  return [{
    id: crypto.randomUUID(),
    type: "OUTBOX_FLUSH",
    payload: { entryIds: outboxEntries.map((e) => e.id) },
    scheduledAt: Date.now(),
  }];
}



// ========================================================================
// STOS V2.3.2 — KV STORE
// All Deno KV access goes through this module.
// Namespace keys mirror the storage hierarchy in Section 4.
// ========================================================================

import type {
  UserRecord,
  GroupRecord,
  ChannelRecord,
  PostRecord,
  TicketRecord,
  PollRecord,
  TopicRecord,
  BroadcastRecord,
  ReminderRecord,
  AuditEntry,
  OutboxEntry,
  QueueItem,
  AtomicCommitPayload,
  AggregateUpdate,
  KVEntry,
} from "../types/index.ts";

// -----------------------------------------------------------------------
// KV INSTANCE
// -----------------------------------------------------------------------

let _kv: Deno.Kv | null = null;

export async function getKV(): Promise<Deno.Kv> {
  if (!_kv) {
    _kv = await Deno.openKv();
  }
  return _kv;
}

// -----------------------------------------------------------------------
// KEY BUILDERS (Section 4 namespace hierarchy)
// -----------------------------------------------------------------------

export const Keys = {
  // system/*
  system: (field: string): Deno.KvKey => ["system", field],

  // owner/*
  owner: (field: string): Deno.KvKey => ["owner", field],

  // users/*
  user: (userId: number): Deno.KvKey => ["users", userId],
  userFsm: (userId: number): Deno.KvKey => ["users", userId, "fsm"],

  // groups/*
  group: (groupId: number): Deno.KvKey => ["groups", groupId],

  // channels/*
  channel: (channelId: number): Deno.KvKey => ["channels", channelId],

  // posts/*
  postDraft: (postId: string): Deno.KvKey => ["posts", "drafts", postId],
  postScheduled: (postId: string): Deno.KvKey => ["posts", "scheduled", postId],
  postPublished: (postId: string): Deno.KvKey => ["posts", "published", postId],
  postArchived: (postId: string): Deno.KvKey => ["posts", "archived", postId],

  // community/*
  communityJoinRequest: (userId: number): Deno.KvKey => ["community", "join_requests", userId],

  // tickets/*
  ticket: (ticketId: string): Deno.KvKey => ["tickets", ticketId],
  ticketByUser: (userId: number): Deno.KvKey => ["tickets", "by_user", userId],

  // polls/*
  poll: (pollId: string): Deno.KvKey => ["polls", pollId],

  // topics/*
  topic: (topicId: string): Deno.KvKey => ["topics", topicId],

  // broadcasts/*
  broadcast: (broadcastId: string): Deno.KvKey => ["broadcasts", broadcastId],

  // reminders/*
  reminder: (reminderId: string): Deno.KvKey => ["reminders", reminderId],
  remindersByUser: (userId: number): Deno.KvKey => ["reminders", "by_user", userId],

  // scheduler/*
  schedulerIndex: (): Deno.KvKey => ["scheduler", "index"],

  // audit/*
  audit: (auditId: string): Deno.KvKey => ["audit", auditId],

  // aggregates/*
  aggregate: (metric: string): Deno.KvKey => ["aggregates", metric],

  // outbox/*
  outbox: (entryId: string): Deno.KvKey => ["outbox", entryId],

  // queue/*
  queue: (itemId: string): Deno.KvKey => ["queue", itemId],

  // locks/*
  lock: (resource: string): Deno.KvKey => ["locks", resource],

  // idempotency/*
  idempotency: (updateId: number): Deno.KvKey => ["idempotency", updateId],
};

// -----------------------------------------------------------------------
// ATOMIC COMMIT (Section 5)
// All five outputs committed in one transaction. No partial writes.
// -----------------------------------------------------------------------

export async function atomicCommit(payload: AtomicCommitPayload): Promise<boolean> {
  const kv = await getKV();
  let tx = kv.atomic();

  // 1. State updates
  for (const entry of payload.stateUpdates) {
    if (entry.versionstamp) {
      tx = tx.check({ key: entry.key, versionstamp: entry.versionstamp });
    }
    tx = tx.set(entry.key, entry.value);
  }

  // 2. Audit log entry (immutable — INV-07)
  tx = tx.set(Keys.audit(payload.auditEntry.id), payload.auditEntry);

  // 3. Aggregate updates
  for (const agg of payload.aggregateUpdates) {
    tx = tx.sum(agg.key, BigInt(agg.delta));
  }

  // 4. Outbox entries
  for (const entry of payload.outboxEntries) {
    tx = tx.set(Keys.outbox(entry.id), entry);
  }

  // 5. Queue items
  for (const item of payload.queueItems) {
    tx = tx.enqueue(item, { delay: Math.max(0, item.scheduledAt - Date.now()) });
    tx = tx.set(Keys.queue(item.id), item);
  }

  const result = await tx.commit();
  return result.ok;
}

// -----------------------------------------------------------------------
// IDEMPOTENCY (Step 12)
// -----------------------------------------------------------------------

export async function isAlreadyProcessed(updateId: number): Promise<boolean> {
  const kv = await getKV();
  const entry = await kv.get(Keys.idempotency(updateId));
  return entry.value !== null;
}

export async function markProcessed(updateId: number): Promise<void> {
  const kv = await getKV();
  await kv.set(Keys.idempotency(updateId), { processedAt: Date.now() }, {
    expireIn: 1000 * 60 * 60 * 24 * 7, // 7 days
  });
}

// -----------------------------------------------------------------------
// LOCK MANAGEMENT (Step 13)
// -----------------------------------------------------------------------

const LOCK_TTL_MS = 10_000;

export async function acquireLock(resource: string): Promise<string | null> {
  const kv = await getKV();
  const lockId = crypto.randomUUID();
  const key = Keys.lock(resource);

  const existing = await kv.get(key);
  if (existing.value !== null) return null;

  const result = await kv
    .atomic()
    .check({ key, versionstamp: null })
    .set(key, { lockId, acquiredAt: Date.now() }, { expireIn: LOCK_TTL_MS })
    .commit();

  return result.ok ? lockId : null;
}

export async function releaseLock(resource: string, lockId: string): Promise<void> {
  const kv = await getKV();
  const key = Keys.lock(resource);
  const existing = await kv.get<{ lockId: string }>(key);
  if (existing.value?.lockId === lockId) {
    await kv.delete(key);
  }
}

// -----------------------------------------------------------------------
// USER CRUD
// -----------------------------------------------------------------------

export async function getUser(userId: number): Promise<UserRecord | null> {
  const kv = await getKV();
  const entry = await kv.get<UserRecord>(Keys.user(userId));
  return entry.value;
}

export async function upsertUser(record: UserRecord): Promise<KVEntry> {
  const kv = await getKV();
  const existing = await kv.get(Keys.user(record.id));
  return {
    key: Keys.user(record.id),
    value: record,
    versionstamp: existing.versionstamp ?? undefined,
  };
}

// -----------------------------------------------------------------------
// GROUP CRUD
// -----------------------------------------------------------------------

export async function getGroup(groupId: number): Promise<GroupRecord | null> {
  const kv = await getKV();
  const entry = await kv.get<GroupRecord>(Keys.group(groupId));
  return entry.value;
}

export async function upsertGroup(record: GroupRecord): Promise<KVEntry> {
  const kv = await getKV();
  const existing = await kv.get(Keys.group(record.id));
  return {
    key: Keys.group(record.id),
    value: record,
    versionstamp: existing.versionstamp ?? undefined,
  };
}

// -----------------------------------------------------------------------
// CHANNEL CRUD
// -----------------------------------------------------------------------

export async function getChannel(channelId: number): Promise<ChannelRecord | null> {
  const kv = await getKV();
  const entry = await kv.get<ChannelRecord>(Keys.channel(channelId));
  return entry.value;
}

export async function upsertChannel(record: ChannelRecord): Promise<KVEntry> {
  const kv = await getKV();
  const existing = await kv.get(Keys.channel(record.id));
  return {
    key: Keys.channel(record.id),
    value: record,
    versionstamp: existing.versionstamp ?? undefined,
  };
}

// -----------------------------------------------------------------------
// POST CRUD
// -----------------------------------------------------------------------

export function postKey(post: PostRecord): Deno.KvKey {
  switch (post.status) {
    case "DRAFT": return Keys.postDraft(post.id);
    case "SCHEDULED": return Keys.postScheduled(post.id);
    case "PUBLISHED": return Keys.postPublished(post.id);
    case "ARCHIVED": return Keys.postArchived(post.id);
  }
}

export async function getPost(postId: string, status: PostRecord["status"]): Promise<PostRecord | null> {
  const kv = await getKV();
  const key = status === "DRAFT" ? Keys.postDraft(postId)
    : status === "SCHEDULED" ? Keys.postScheduled(postId)
    : status === "PUBLISHED" ? Keys.postPublished(postId)
    : Keys.postArchived(postId);
  const entry = await kv.get<PostRecord>(key);
  return entry.value;
}

export function buildPostEntry(post: PostRecord): KVEntry {
  return { key: postKey(post), value: post };
}

// -----------------------------------------------------------------------
// TICKET CRUD
// -----------------------------------------------------------------------

export async function getTicket(ticketId: string): Promise<TicketRecord | null> {
  const kv = await getKV();
  const entry = await kv.get<TicketRecord>(Keys.ticket(ticketId));
  return entry.value;
}

export async function getActiveTicketForUser(userId: number): Promise<TicketRecord | null> {
  const kv = await getKV();
  const ref = await kv.get<string>(Keys.ticketByUser(userId));
  if (!ref.value) return null;
  const entry = await kv.get<TicketRecord>(Keys.ticket(ref.value));
  return entry.value;
}

export function buildTicketEntries(ticket: TicketRecord): KVEntry[] {
  return [
    { key: Keys.ticket(ticket.id), value: ticket },
    { key: Keys.ticketByUser(ticket.userId), value: ticket.id },
  ];
}

// -----------------------------------------------------------------------
// POLL CRUD
// -----------------------------------------------------------------------

export async function getPoll(pollId: string): Promise<PollRecord | null> {
  const kv = await getKV();
  const entry = await kv.get<PollRecord>(Keys.poll(pollId));
  return entry.value;
}

export function buildPollEntry(poll: PollRecord): KVEntry {
  return { key: Keys.poll(poll.id), value: poll };
}

// -----------------------------------------------------------------------
// BROADCAST CRUD
// -----------------------------------------------------------------------

export async function getBroadcast(broadcastId: string): Promise<BroadcastRecord | null> {
  const kv = await getKV();
  const entry = await kv.get<BroadcastRecord>(Keys.broadcast(broadcastId));
  return entry.value;
}

export function buildBroadcastEntry(broadcast: BroadcastRecord): KVEntry {
  return { key: Keys.broadcast(broadcast.id), value: broadcast };
}

// -----------------------------------------------------------------------
// REMINDER CRUD
// -----------------------------------------------------------------------

export async function getReminder(reminderId: string): Promise<ReminderRecord | null> {
  const kv = await getKV();
  const entry = await kv.get<ReminderRecord>(Keys.reminder(reminderId));
  return entry.value;
}

export function buildReminderEntries(reminder: ReminderRecord): KVEntry[] {
  return [
    { key: Keys.reminder(reminder.id), value: reminder },
    { key: Keys.remindersByUser(reminder.userId), value: reminder.id },
  ];
}

// -----------------------------------------------------------------------
// SYSTEM CONFIG
// -----------------------------------------------------------------------

export async function getSystemConfig(field: string): Promise<unknown> {
  const kv = await getKV();
  const entry = await kv.get(Keys.system(field));
  return entry.value;
}

export async function setSystemConfig(field: string, value: unknown): Promise<void> {
  const kv = await getKV();
  await kv.set(Keys.system(field), value);
}

// -----------------------------------------------------------------------
// OUTBOX LISTING
// -----------------------------------------------------------------------

export async function listPendingOutbox(): Promise<OutboxEntry[]> {
  const kv = await getKV();
  const entries: OutboxEntry[] = [];
  const iter = kv.list<OutboxEntry>({ prefix: ["outbox"] });
  for await (const entry of iter) {
    if (entry.value && entry.value.nextAttemptAt <= Date.now()) {
      entries.push(entry.value);
    }
  }
  return entries;
}

export async function deleteOutboxEntry(entryId: string): Promise<void> {
  const kv = await getKV();
  await kv.delete(Keys.outbox(entryId));
}

export async function updateOutboxEntry(entry: OutboxEntry): Promise<void> {
  const kv = await getKV();
  await kv.set(Keys.outbox(entry.id), entry);
}



