# Worked examples: from code to spec

These examples show real implementations in Python and TypeScript, then walk through extracting equivalent Recife model components.

## Example 1: Password Reset (Python/Flask)

**The implementation:**

```python
# models.py
from datetime import datetime, timedelta
from werkzeug.security import generate_password_hash, check_password_hash
import secrets

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(256), nullable=False)
    status = db.Column(db.String(20), default='active')
    failed_attempts = db.Column(db.Integer, default=0)
    locked_until = db.Column(db.DateTime, nullable=True)

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

    def is_locked(self):
        return (self.status == 'locked' and
                self.locked_until and
                self.locked_until > datetime.utcnow())


class PasswordResetToken(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    token = db.Column(db.String(64), unique=True, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    expires_at = db.Column(db.DateTime, nullable=False)
    used = db.Column(db.Boolean, default=False)

    user = db.relationship('User', backref='reset_tokens')

    @staticmethod
    def generate_token():
        return secrets.token_urlsafe(32)

    def is_valid(self):
        return (not self.used and
                self.expires_at > datetime.utcnow())


# routes.py
from flask import request, jsonify
from flask_mail import Message

RESET_TOKEN_EXPIRY_HOURS = 1
MAX_FAILED_ATTEMPTS = 5
LOCKOUT_MINUTES = 15

@app.route('/api/auth/request-reset', methods=['POST'])
def request_password_reset():
    data = request.get_json()
    email = data.get('email')

    user = User.query.filter_by(email=email).first()
    if not user:
        # Return success anyway to prevent email enumeration
        return jsonify({'message': 'If account exists, reset email sent'}), 200

    if user.status == 'deactivated':
        return jsonify({'message': 'If account exists, reset email sent'}), 200

    # Invalidate existing tokens
    PasswordResetToken.query.filter_by(
        user_id=user.id,
        used=False
    ).update({'used': True})

    # Create new token
    token = PasswordResetToken(
        user_id=user.id,
        token=PasswordResetToken.generate_token(),
        expires_at=datetime.utcnow() + timedelta(hours=RESET_TOKEN_EXPIRY_HOURS)
    )
    db.session.add(token)
    db.session.commit()

    # Send email
    reset_url = f"{app.config['FRONTEND_URL']}/reset-password?token={token.token}"
    msg = Message(
        'Password Reset Request',
        recipients=[user.email],
        html=render_template('emails/password_reset.html',
                           user=user,
                           reset_url=reset_url)
    )
    mail.send(msg)

    return jsonify({'message': 'If account exists, reset email sent'}), 200


@app.route('/api/auth/reset-password', methods=['POST'])
def reset_password():
    data = request.get_json()
    token_string = data.get('token')
    new_password = data.get('password')

    if len(new_password) < 12:
        return jsonify({'error': 'Password must be at least 12 characters'}), 400

    token = PasswordResetToken.query.filter_by(token=token_string).first()

    if not token or not token.is_valid():
        return jsonify({'error': 'Invalid or expired token'}), 400

    user = token.user

    # Mark token as used
    token.used = True

    # Update password
    user.set_password(new_password)
    user.status = 'active'
    user.failed_attempts = 0
    user.locked_until = None

    # Invalidate all sessions (assuming Session model exists)
    Session.query.filter_by(
        user_id=user.id,
        status='active'
    ).update({'status': 'revoked'})

    db.session.commit()

    # Send confirmation email
    msg = Message(
        'Password Changed',
        recipients=[user.email],
        html=render_template('emails/password_changed.html', user=user)
    )
    mail.send(msg)

    return jsonify({'message': 'Password reset successful'}), 200


# Scheduled job (e.g., celery task)
@celery.task
def cleanup_expired_tokens():
    """Run hourly to mark expired tokens"""
    PasswordResetToken.query.filter(
        PasswordResetToken.used == False,
        PasswordResetToken.expires_at < datetime.utcnow()
    ).update({'used': True})
    db.session.commit()
```

**Extraction process:**

1. **Identify entities from models:**
   - `User` - has email, password_hash, status, failed_login_attempts, locked_until
   - `PasswordResetToken` - has user, token, created_at, expires_at, used

2. **Identify states from status fields and booleans:**
   - User status: `active | locked | deactivated` (found in code)
   - Token: `used` boolean, convert to status: `pending | used | expired`

3. **Identify triggers from routes/handlers:**
   - `request_password_reset` - external trigger
   - `reset_password` - external trigger
   - `cleanup_expired_tokens` - temporal trigger

4. **Extract preconditions from validation:**
   - `if not user` becomes `requires: exists user`
   - `len(new_password) < 12` becomes `requires: length(password) >= 12`
   - `token.is_valid()` becomes `requires: token.is_valid`

5. **Extract postconditions from mutations:**
   - `token.used = True` becomes `ensures: token.status = used`
   - `user.set_password(...)` becomes `ensures: user.password_hash = hash(password)`
   - `mail.send(msg)` becomes `ensures: Email.created(...)`

6. **Strip implementation details:**
   - Remove: `secrets.token_urlsafe(32)`, `generate_password_hash`, `db.session`
   - Remove: HTTP status codes, JSON responses
   - Remove: `render_template`, URL construction
   - Keep: durations (1 hour, 12 characters)

**Extracted Recife model:**

```clojure
;; password-reset.model.clj
(ns worked.password-reset.model
  (:require [recife.core :as r]))

(def global
  {::config {:reset-token-expiry-ms (* 60 60 1000) ; 1.hour
             :min-password-length 12}
   ::users {}
   ::reset-tokens {}
   ::sessions {}
   :auth/request-password-reset #{}
   :auth/complete-password-reset #{}
   :mail/outbox []
   :clock/now 0})

(defn token-valid? [token now]
  (and (= :pending (:status token))
       (> (:expires-at token) now)))

(defn hash-password [raw]
  (str "hash:" raw))
(r/defproc request-password-reset
  (fn [{:keys [:auth/request-password-reset ::users ::reset-tokens ::config :clock/now] :as db}]
    (when-let [{:keys [email]} (first request-password-reset)]
      (when-let [[user-id user] (first (filter (fn [[_ u]] (= email (:email u))) users))]
        (when (contains? #{:active :locked} (:status user))
          (let [token-id (keyword (str "token-" (inc (count reset-tokens))))
                db' (-> db
                        (update :auth/request-password-reset disj {:email email})
                        (update ::reset-tokens
                                (fn [tokens]
                                  (into {}
                                        (map (fn [[tid token]]
                                               [tid (if (and (= user-id (:user-id token))
                                                             (= :pending (:status token)))
                                                      (assoc token :status :expired)
                                                      token)]))
                                        tokens)))
                        (assoc-in [::reset-tokens token-id]
                                  {:user-id user-id
                                   :created-at now
                                   :expires-at (+ now (get-in db [::config :reset-token-expiry-ms]))
                                   :status :pending}))]
            (update db' :mail/outbox conj
                    {:to (:email user)
                     :template :password-reset
                     :data {:token-id token-id}})))))))
(r/defproc complete-password-reset
  (fn [{:keys [:auth/complete-password-reset ::reset-tokens ::users ::sessions ::config :clock/now] :as db}]
    (when-let [{:keys [token-id new-password]} (first complete-password-reset)]
      (let [token (get reset-tokens token-id)]
        (when (and token
                   (token-valid? token now)
                   (>= (count new-password) (get-in config [:min-password-length])))
          (let [user-id (:user-id token)
                user (get users user-id)
                db' (-> db
                        (update :auth/complete-password-reset disj {:token-id token-id :new-password new-password})
                        (assoc-in [::reset-tokens token-id :status] :used)
                        (assoc-in [::users user-id :password-hash] (hash-password new-password))
                        (assoc-in [::users user-id :status] :active)
                        (assoc-in [::users user-id :failed-login-attempts] 0)
                        (assoc-in [::users user-id :locked-until] nil)
                        (update ::sessions
                                (fn [all-sessions]
                                  (into {}
                                        (map (fn [[sid session]]
                                               [sid (if (and (= user-id (:user-id session))
                                                             (= :active (:status session)))
                                                      (assoc session :status :revoked)
                                                      session)]))
                                        all-sessions))))]
            (update db' :mail/outbox conj
                    {:to (:email user)
                     :template :password-changed})))))))
(r/defproc reset-token-expires
  (fn [{:keys [::reset-tokens :clock/now] :as db}]
    (reduce-kv (fn [acc token-id token]
                 (if (and (= :pending (:status token))
                          (<= (:expires-at token) now))
                   (assoc-in acc [::reset-tokens token-id :status] :expired)
                   acc))
               db
               reset-tokens)))
```

**What we removed:**
- Database details (SQLAlchemy, column types, foreign keys)
- HTTP layer (routes, JSON, status codes)
- Security implementation (token generation algorithm, password hashing)
- Email enumeration protection (design decision, could add back if desired)
- Template rendering details

---

## Example 2: Usage Limits (TypeScript/Node)

**The implementation:**

```typescript
// models/plan.ts
export interface Plan {
  id: string;
  name: string;
  maxProjects: number;      // -1 for unlimited
  maxStorageMB: number;     // -1 for unlimited
  maxTeamMembers: number;
  monthlyPriceUsd: number;
  features: string[];
}

export const PLANS: Record<string, Plan> = {
  free: {
    id: 'free',
    name: 'Free',
    maxProjects: 3,
    maxStorageMB: 100,
    maxTeamMembers: 1,
    monthlyPriceUsd: 0,
    features: ['basic_editor'],
  },
  pro: {
    id: 'pro',
    name: 'Pro',
    maxProjects: 50,
    maxStorageMB: 10000,
    maxTeamMembers: 10,
    monthlyPriceUsd: 15,
    features: ['basic_editor', 'advanced_editor', 'api_access'],
  },
  enterprise: {
    id: 'enterprise',
    name: 'Enterprise',
    maxProjects: -1,
    maxStorageMB: -1,
    maxTeamMembers: -1,
    monthlyPriceUsd: 99,
    features: ['basic_editor', 'advanced_editor', 'api_access', 'sso', 'audit_log'],
  },
};

// models/workspace.ts
export interface Workspace {
  id: string;
  name: string;
  ownerId: string;
  planId: string;
  createdAt: Date;
}

// services/usage.service.ts
import { prisma } from '../db';
import { PLANS } from '../models/plan';

export class UsageService {
  async getWorkspaceUsage(workspaceId: string) {
    const [projectCount, storageBytes, memberCount] = await Promise.all([
      prisma.project.count({ where: { workspaceId, deletedAt: null } }),
      prisma.file.aggregate({
        where: { project: { workspaceId } },
        _sum: { sizeBytes: true },
      }),
      prisma.workspaceMember.count({ where: { workspaceId } }),
    ]);

    return {
      projects: projectCount,
      storageMB: Math.ceil((storageBytes._sum.sizeBytes || 0) / 1024 / 1024),
      members: memberCount,
    };
  }

  async canCreateProject(workspaceId: string): Promise<boolean> {
    const workspace = await prisma.workspace.findUnique({
      where: { id: workspaceId },
    });
    if (!workspace) return false;

    const plan = PLANS[workspace.planId];
    if (plan.maxProjects === -1) return true;

    const usage = await this.getWorkspaceUsage(workspaceId);
    return usage.projects < plan.maxProjects;
  }

  async canAddMember(workspaceId: string): Promise<boolean> {
    const workspace = await prisma.workspace.findUnique({
      where: { id: workspaceId },
    });
    if (!workspace) return false;

    const plan = PLANS[workspace.planId];
    if (plan.maxTeamMembers === -1) return true;

    const usage = await this.getWorkspaceUsage(workspaceId);
    return usage.members < plan.maxTeamMembers;
  }

  async canUploadFile(workspaceId: string, fileSizeBytes: number): Promise<boolean> {
    const workspace = await prisma.workspace.findUnique({
      where: { id: workspaceId },
    });
    if (!workspace) return false;

    const plan = PLANS[workspace.planId];
    if (plan.maxStorageMB === -1) return true;

    const usage = await this.getWorkspaceUsage(workspaceId);
    const newStorageMB = usage.storageMB + Math.ceil(fileSizeBytes / 1024 / 1024);
    return newStorageMB <= plan.maxStorageMB;
  }

  hasFeature(planId: string, feature: string): boolean {
    const plan = PLANS[planId];
    return plan?.features.includes(feature) ?? false;
  }
}

// controllers/project.controller.ts
import { UsageService } from '../services/usage.service';

const usageService = new UsageService();

export async function createProject(req: Request, res: Response) {
  const { workspaceId, name } = req.body;
  const userId = req.user.id;

  // Check membership
  const membership = await prisma.workspaceMember.findUnique({
    where: { workspaceId_userId: { workspaceId, userId } },
  });

  if (!membership) {
    return res.status(403).json({ error: 'Not a member of this workspace' });
  }

  // Check limits
  const canCreate = await usageService.canCreateProject(workspaceId);
  if (!canCreate) {
    const workspace = await prisma.workspace.findUnique({
      where: { id: workspaceId },
      include: { plan: true },
    });

    return res.status(403).json({
      error: 'Project limit reached',
      code: 'LIMIT_REACHED',
      limit: PLANS[workspace!.planId].maxProjects,
      upgradeUrl: '/settings/billing',
    });
  }

  const project = await prisma.project.create({
    data: {
      workspaceId,
      name,
      createdById: userId,
    },
  });

  // Track usage event
  await prisma.usageEvent.create({
    data: {
      workspaceId,
      type: 'PROJECT_CREATED',
      metadata: { projectId: project.id },
    },
  });

  return res.status(201).json(project);
}

// controllers/billing.controller.ts
export async function changePlan(req: Request, res: Response) {
  const { workspaceId, newPlanId } = req.body;
  const userId = req.user.id;

  const workspace = await prisma.workspace.findUnique({
    where: { id: workspaceId },
  });

  if (!workspace || workspace.ownerId !== userId) {
    return res.status(403).json({ error: 'Only owner can change plan' });
  }

  const currentPlan = PLANS[workspace.planId];
  const newPlan = PLANS[newPlanId];

  if (!newPlan) {
    return res.status(400).json({ error: 'Invalid plan' });
  }

  // Check if downgrading
  const isDowngrade = newPlan.monthlyPriceUsd < currentPlan.monthlyPriceUsd;

  if (isDowngrade) {
    const usage = await usageService.getWorkspaceUsage(workspaceId);

    // Validate limits
    if (newPlan.maxProjects !== -1 && usage.projects > newPlan.maxProjects) {
      return res.status(400).json({
        error: 'Cannot downgrade: too many projects',
        code: 'DOWNGRADE_BLOCKED',
        current: usage.projects,
        limit: newPlan.maxProjects,
        mustDelete: usage.projects - newPlan.maxProjects,
      });
    }

    if (newPlan.maxStorageMB !== -1 && usage.storageMB > newPlan.maxStorageMB) {
      return res.status(400).json({
        error: 'Cannot downgrade: storage exceeds limit',
        code: 'DOWNGRADE_BLOCKED',
        currentMB: usage.storageMB,
        limitMB: newPlan.maxStorageMB,
      });
    }

    if (newPlan.maxTeamMembers !== -1 && usage.members > newPlan.maxTeamMembers) {
      return res.status(400).json({
        error: 'Cannot downgrade: too many team members',
        code: 'DOWNGRADE_BLOCKED',
        current: usage.members,
        limit: newPlan.maxTeamMembers,
      });
    }
  }

  await prisma.workspace.update({
    where: { id: workspaceId },
    data: { planId: newPlanId },
  });

  // Send email notification
  const owner = await prisma.user.findUnique({ where: { id: workspace.ownerId } });
  await sendEmail({
    to: owner!.email,
    template: isDowngrade ? 'plan_downgraded' : 'plan_upgraded',
    data: { oldPlan: currentPlan.name, newPlan: newPlan.name },
  });

  return res.json({ success: true, plan: newPlan });
}
```

**Extraction process:**

1. **Identify entities from types/models:**
   - `Plan` - configuration entity with limits
   - `Workspace` - has owner, plan
   - `WorkspaceMembership` - join entity (user + workspace)
   - `Project`, `File` - resources that count against limits
   - `UsageEvent` - audit/tracking

2. **Identify derived values from service methods:**
   - `canCreateProject()` becomes a derived boolean on Workspace
   - `canAddMember()` becomes a derived boolean
   - `hasFeature()` becomes a derived function

3. **Recognize the "unlimited" pattern:**
   - `-1` means unlimited, convert to explicit handling

4. **Identify triggers from controllers:**
   - `createProject` - external trigger with limit check
   - `changePlan` - external trigger with downgrade validation

5. **Extract the permission/limit pattern:**
   - Check membership becomes `requires: exists membership`
   - Check limit becomes `requires: workspace.can_add_project`
   - Return error with upgrade path becomes a separate rule for limit reached

**Extracted Recife model:**

```clojure
;; usage-limits.model.clj
(ns worked.usage-limits.model
  (:require [recife.core :as r]))

(def global
  {::plans {}
   ::workspaces {}
   ::workspace-memberships {} ; [workspace-id user-id] -> {:can-admin true/false}
   ::projects {}
   ::usage-events []
   :workspace/create-project #{}
   :workspace/change-plan #{}
   :notifications/outbox []
   :mail/outbox []})

(defn calculate-storage-mb [_db _workspace-id]
  0)

(defn workspace-project-count [db workspace-id]
  (->> (::projects db)
       vals
       (filter #(and (= workspace-id (:workspace-id %))
                     (nil? (:deleted-at %))))
       count))

(defn workspace-member-count [db workspace-id]
  (->> (::workspace-memberships db)
       keys
       (filter (fn [[wid _]] (= workspace-id wid)))
       count))

(defn has-unlimited? [n]
  (= -1 n))

(defn can-add-project? [db workspace]
  (let [plan (get-in db [::plans (:plan-id workspace)])
        limit (:max-projects plan)]
    (or (has-unlimited? limit)
        (< (workspace-project-count db (:id workspace)) limit))))
(r/defproc create-project
  (fn [{:keys [:workspace/create-project ::workspaces ::workspace-memberships] :as db}]
    (when-let [{:keys [user-id workspace-id name]} (first create-project)]
      (let [workspace (get workspaces workspace-id)
            membership (get workspace-memberships [workspace-id user-id])]
        (when (and workspace membership (can-add-project? db workspace))
          (let [project-id (keyword (str "project-" (inc (count (::projects db)))))]
            (-> db
                (update :workspace/create-project disj {:user-id user-id :workspace-id workspace-id :name name})
                (assoc-in [::projects project-id]
                          {:workspace-id workspace-id
                           :name name
                           :created-by user-id
                           :deleted-at nil})
                (update ::usage-events conj {:workspace-id workspace-id
                                             :type :project-created}))))))))
(r/defproc create-project-limit-reached
  (fn [{:keys [:workspace/create-project ::workspaces ::workspace-memberships] :as db}]
    (when-let [{:keys [user-id workspace-id name] :as cmd} (first create-project)]
      (let [workspace (get workspaces workspace-id)
            membership (get workspace-memberships [workspace-id user-id])
            plan (get-in db [::plans (:plan-id workspace)])]
        (when (and workspace membership (not (can-add-project? db workspace)))
          (-> db
              (update :workspace/create-project disj cmd)
              (update :notifications/outbox conj
                      {:user-id user-id
                       :about :limit-reached
                       :data {:limit-type :projects
                              :current (workspace-project-count db workspace-id)
                              :max (:max-projects plan)
                              :name name}})))))))
(r/defproc change-plan
  (fn [{:keys [:workspace/change-plan ::workspaces ::plans] :as db}]
    (when-let [{:keys [user-id workspace-id new-plan-id] :as cmd} (first change-plan)]
      (let [workspace (get workspaces workspace-id)
            old-plan (some->> workspace :plan-id (get plans))
            new-plan (get plans new-plan-id)
            is-downgrade (< (:monthly-price new-plan) (:monthly-price old-plan))
            project-ok (or (not is-downgrade)
                           (has-unlimited? (:max-projects new-plan))
                           (<= (workspace-project-count db workspace-id) (:max-projects new-plan)))
            storage-ok (or (not is-downgrade)
                           (has-unlimited? (:max-storage-mb new-plan))
                           (<= (calculate-storage-mb db workspace-id) (:max-storage-mb new-plan)))
            members-ok (or (not is-downgrade)
                           (has-unlimited? (:max-team-members new-plan))
                           (<= (workspace-member-count db workspace-id) (:max-team-members new-plan)))]
        (when (and workspace
                   (= user-id (:owner-id workspace))
                   project-ok
                   storage-ok
                   members-ok)
          (-> db
              (update :workspace/change-plan disj cmd)
              (assoc-in [::workspaces workspace-id :plan-id] new-plan-id)
              (update :mail/outbox conj
                      {:to (:owner-email workspace)
                       :template (if is-downgrade :plan-downgraded :plan-upgraded)
                       :data {:old-plan old-plan :new-plan new-plan}})))))))
(r/defproc downgrade-blocked
  (fn [{:keys [:workspace/change-plan ::workspaces ::plans] :as db}]
    (when-let [{:keys [user-id workspace-id new-plan-id] :as cmd} (first change-plan)]
      (let [workspace (get workspaces workspace-id)
            old-plan (some->> workspace :plan-id (get plans))
            new-plan (get plans new-plan-id)
            is-downgrade (< (:monthly-price new-plan) (:monthly-price old-plan))
            project-count (workspace-project-count db workspace-id)
            project-limit (:max-projects new-plan)]
        (when (and workspace
                   (= user-id (:owner-id workspace))
                   is-downgrade
                   (not (has-unlimited? project-limit))
                   (> project-count project-limit))
          (-> db
              (update :workspace/change-plan disj cmd)
              (update :notifications/outbox conj
                      {:user-id user-id
                       :about :downgrade-blocked
                       :data {:reason :projects
                              :current project-count
                              :limit project-limit}})))))))
```

**What we removed:**
- Prisma queries and database access patterns
- HTTP layer (Express req/res, status codes)
- Promise.all parallelisation
- Math.ceil for storage calculation
- JSON error response structure
- Compound unique key syntax

**What we kept:**
- The -1 unlimited convention (could also use explicit `unlimited` type)
- Plan structure with features
- The paired success/failure rule pattern
- Usage event tracking

---

## Example 3: Soft Delete (Java/Spring)

**The implementation:**

```java
// entities/Document.java
@Entity
@Table(name = "documents")
@Where(clause = "deleted_at IS NULL")  // Default filter
public class Document {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String content;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "workspace_id", nullable = false)
    private Workspace workspace;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "created_by_id", nullable = false)
    private User createdBy;

    @Column(nullable = false)
    private Instant createdAt;

    @Column
    private Instant deletedAt;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "deleted_by_id")
    private User deletedBy;

    public boolean isDeleted() {
        return deletedAt != null;
    }

    public boolean canRestore() {
        if (deletedAt == null) return false;
        Instant retentionDeadline = deletedAt.plus(Duration.ofDays(30));
        return Instant.now().isBefore(retentionDeadline);
    }
}

// repositories/DocumentRepository.java
public interface DocumentRepository extends JpaRepository<Document, String> {

    // This ignores the @Where clause to include deleted documents
    @Query("SELECT d FROM Document d WHERE d.workspace.id = :workspaceId")
    List<Document> findAllIncludingDeleted(@Param("workspaceId") String workspaceId);

    @Query("SELECT d FROM Document d WHERE d.workspace.id = :workspaceId AND d.deletedAt IS NOT NULL")
    List<Document> findDeleted(@Param("workspaceId") String workspaceId);

    @Query("SELECT d FROM Document d WHERE d.workspace.id = :workspaceId AND d.deletedAt IS NOT NULL AND d.deletedAt > :cutoff")
    List<Document> findRestorable(@Param("workspaceId") String workspaceId, @Param("cutoff") Instant cutoff);

    @Modifying
    @Query("DELETE FROM Document d WHERE d.deletedAt IS NOT NULL AND d.deletedAt < :cutoff")
    int permanentlyDeleteExpired(@Param("cutoff") Instant cutoff);
}

// services/DocumentService.java
@Service
@Transactional
public class DocumentService {

    private static final Duration RETENTION_PERIOD = Duration.ofDays(30);

    @Autowired
    private DocumentRepository documentRepository;

    @Autowired
    private WorkspaceMemberRepository memberRepository;

    public void softDelete(String documentId, String userId) {
        Document document = documentRepository.findById(documentId)
            .orElseThrow(() -> new NotFoundException("Document not found"));

        if (document.isDeleted()) {
            throw new IllegalStateException("Document already deleted");
        }

        // Check permission: creator or admin
        boolean isCreator = document.getCreatedBy().getId().equals(userId);
        boolean isAdmin = memberRepository.isAdmin(document.getWorkspace().getId(), userId);

        if (!isCreator && !isAdmin) {
            throw new ForbiddenException("Not authorized to delete this document");
        }

        document.setDeletedAt(Instant.now());
        document.setDeletedBy(userRepository.findById(userId).orElseThrow());

        documentRepository.save(document);
    }

    public void restore(String documentId, String userId) {
        // Bypass @Where to find deleted document
        Document document = documentRepository.findAllIncludingDeleted(documentId)
            .stream()
            .filter(d -> d.getId().equals(documentId))
            .findFirst()
            .orElseThrow(() -> new NotFoundException("Document not found"));

        if (!document.canRestore()) {
            throw new IllegalStateException("Document cannot be restored");
        }

        // Check permission: original deleter or admin
        boolean isDeleter = document.getDeletedBy().getId().equals(userId);
        boolean isAdmin = memberRepository.isAdmin(document.getWorkspace().getId(), userId);

        if (!isDeleter && !isAdmin) {
            throw new ForbiddenException("Not authorized to restore this document");
        }

        document.setDeletedAt(null);
        document.setDeletedBy(null);

        documentRepository.save(document);
    }

    public void permanentlyDelete(String documentId, String userId) {
        Document document = documentRepository.findAllIncludingDeleted(documentId)
            .stream()
            .filter(d -> d.getId().equals(documentId))
            .findFirst()
            .orElseThrow(() -> new NotFoundException("Document not found"));

        if (!document.isDeleted()) {
            throw new IllegalStateException("Document must be soft-deleted first");
        }

        boolean isAdmin = memberRepository.isAdmin(document.getWorkspace().getId(), userId);
        if (!isAdmin) {
            throw new ForbiddenException("Only admins can permanently delete");
        }

        documentRepository.delete(document);
    }

    public void emptyTrash(String workspaceId, String userId) {
        boolean isAdmin = memberRepository.isAdmin(workspaceId, userId);
        if (!isAdmin) {
            throw new ForbiddenException("Only admins can empty trash");
        }

        List<Document> deleted = documentRepository.findDeleted(workspaceId);
        documentRepository.deleteAll(deleted);
    }
}

// scheduled/RetentionCleanupJob.java
@Component
public class RetentionCleanupJob {

    @Autowired
    private DocumentRepository documentRepository;

    @Scheduled(cron = "0 0 2 * * *")  // Run at 2 AM daily
    @Transactional
    public void cleanupExpiredDocuments() {
        Instant cutoff = Instant.now().minus(Duration.ofDays(30));
        int deleted = documentRepository.permanentlyDeleteExpired(cutoff);
        log.info("Permanently deleted {} expired documents", deleted);
    }
}
```

**Extraction process:**

1. **Spot the soft delete pattern:**
   - `deletedAt` timestamp (nullable) instead of status enum
   - `@Where` clause for default filtering
   - Separate queries to include/exclude deleted

2. **Extract the implicit state machine:**
   - `deletedAt = null` means active
   - `deletedAt != null` means deleted
   - `deleted` removes from database, meaning permanently deleted

3. **Identify the retention policy:**
   - `Duration.ofDays(30)` is a config value
   - `canRestore()` method is a derived value

4. **Extract permission rules:**
   - Delete: creator OR admin
   - Restore: original deleter OR admin
   - Permanent delete: admin only

**Extracted Recife model:**

```clojure
;; soft-delete.model.clj
(ns worked.soft-delete.model
  (:require [recife.core :as r]))

(def global
  {::config {:retention-period-ms (* 30 24 60 60 1000)} ; 30.days
   ::documents {}
   ::workspace-memberships {} ; [workspace-id user-id] -> {:can-admin true/false}
   :document/delete #{}
   :document/restore #{}
   :document/permanently-delete #{}
   :workspace/empty-trash #{}
   :clock/now 0})

(defn can-restore? [db document]
  (and (= :deleted (:status document))
       (some? (:deleted-at document))
       (> (+ (:deleted-at document)
             (get-in db [::config :retention-period-ms]))
          (:clock/now db))))

(defn membership-can-admin? [db workspace-id user-id]
  (true? (get-in db [::workspace-memberships [workspace-id user-id] :can-admin])))
(r/defproc delete-document
  (fn [{:keys [:document/delete ::documents] :as db}]
    (when-let [{:keys [actor-id document-id] :as cmd} (first delete)]
      (let [document (get documents document-id)]
        (when (and document
                   (= :active (:status document))
                   (or (= actor-id (:created-by document))
                       (membership-can-admin? db (:workspace-id document) actor-id)))
          (-> db
              (update :document/delete disj cmd)
              (assoc-in [::documents document-id :status] :deleted)
              (assoc-in [::documents document-id :deleted-at] (:clock/now db))
              (assoc-in [::documents document-id :deleted-by] actor-id)))))))
(r/defproc restore-document
  (fn [{:keys [:document/restore ::documents] :as db}]
    (when-let [{:keys [actor-id document-id] :as cmd} (first restore)]
      (let [document (get documents document-id)]
        (when (and document
                   (can-restore? db document)
                   (or (= actor-id (:deleted-by document))
                       (membership-can-admin? db (:workspace-id document) actor-id)))
          (-> db
              (update :document/restore disj cmd)
              (assoc-in [::documents document-id :status] :active)
              (assoc-in [::documents document-id :deleted-at] nil)
              (assoc-in [::documents document-id :deleted-by] nil)))))))
(r/defproc permanently-delete
  (fn [{:keys [:document/permanently-delete ::documents] :as db}]
    (when-let [{:keys [actor-id document-id] :as cmd} (first permanently-delete)]
      (let [document (get documents document-id)]
        (when (and document
                   (= :deleted (:status document))
                   (membership-can-admin? db (:workspace-id document) actor-id))
          (-> db
              (update :document/permanently-delete disj cmd)
              (update ::documents dissoc document-id)))))))
(r/defproc empty-trash
  (fn [{:keys [:workspace/empty-trash ::documents] :as db}]
    (when-let [{:keys [actor-id workspace-id] :as cmd} (first empty-trash)]
      (when (membership-can-admin? db workspace-id actor-id)
        (-> db
            (update :workspace/empty-trash disj cmd)
            (update ::documents
                    (fn [docs]
                      (into {}
                            (remove (fn [[_ doc]]
                                      (and (= workspace-id (:workspace-id doc))
                                           (= :deleted (:status doc)))))
                            docs))))))))
(r/defproc retention-expires
  (fn [{:keys [::documents :clock/now] :as db}]
    (reduce-kv (fn [acc document-id document]
                 (let [retention-expires-at (+ (:deleted-at document)
                                               (get-in db [::config :retention-period-ms]))]
                   (if (and (= :deleted (:status document))
                            (<= retention-expires-at now))
                     (update acc ::documents dissoc document-id)
                     acc)))
               db
               documents)))
```

**Key observations:**

The Java code uses `deletedAt != null` as the delete indicator, but the spec uses an explicit `status` field. Both are valid approaches. The spec is more explicit about state, while the code uses a convention. The spec captures the *meaning* (document is either active or deleted) without prescribing the implementation (status enum vs nullable timestamp).
