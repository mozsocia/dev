### Professional, Scalable & Production-Ready Workflow Automation for Your MERN CRM (Campaign-Centric)

You're building a serious CRM → you need **real workflow automation** that can power lead nurturing, multi-step campaigns, drip sequences, event-based triggers, etc.  
Think HubSpot Workflows / ActiveCampaign Automation / Marketo Smart Campaigns level.

Here’s the **best-practice implementation** in 2024-2025 standards.

### 1. Core Workflow Models (Highly Scalable Design)

```javascript
// models/Workflow.js
const mongoose = require('mongoose');

const WorkflowSchema = new mongoose.Schema({
  name: { type: String, required: true, trim: true },
  description: String,

  campaign: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'Campaign', 
    required: true 
  },

  trigger: {
    type: String,
    enum: [
      'campaign_starts',
      'lead_enters_campaign',     // when a contactily contact is added to campaign
      'form_submitted',
      'tag_added',
      'tag_removed',
      'deal_stage_changed',
      'lead_score_reaches',
      'date_reached',             // absolute date
      'days_after_enrollment',    // relative delay
      'webhook_received',
      'manual_trigger'            // user clicks "Start Workflow" on contact
    ],
    required: true
  },

  triggerConfig: { type: mongoose.Schema.Types.Mixed }, // e.g. { formId, tagId, score: 50, days: 7 }

  isActive: { type: Boolean, default: true },
  runOncePerContact: { type: Boolean, default: true }, // important for nurturing

  enrolledCount: { type: Number, default: 0 },
  completedCount: { type: Number, default: 0 },

  // Visual canvas metadata (for frontend drag-and-drop builder)
  canvasData: { type: mongoose.Schema.Types.Mixed }, // nodes, edges, positions

  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  stats: {
    lastRunAt: Date,
    totalRuns: Number,
    errorCount: Number
  }
}, { timestamps: true });

module.exports = mongoose.model('Workflow', WorkflowSchema);
```

```javascript
// models/WorkflowNode.js  ← This is the key to scalability
const WorkflowNodeSchema = new mongoose.Schema({
  workflow: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'Workflow', 
    required: true 
  },

  nodeId: { type: String, required: true }, // unique within workflow (e.g. "node_1")
  type: {
    type: String,
    enum: [
      'trigger',       // only one per workflow
      'delay',
      'send_email',
      'send_sms',
      'add_tag',
      'remove_tag',
      'update_field',
      'create_task',
      'notify_user',
      'if_else',
      'goal',          // success metric
      'webhook',
      'wait_for_event',
      'exit_workflow'
    ],
    required: true
  },

  config: { type: mongoose.Schema.Types.Mixed, required: true },
  // Examples:
  // delay → { duration: 3, unit: 'days' }
  // send_email → { templateId: '...', fromName: 'John' }
  // if_else → { conditions: [ { field: 'leadScore', operator: 'gte', value: 70 } ] }

  position: { x: Number, y: Number }, // for canvas
  nextNodeId: String,                  // for linear flows
  nextNodes: [{                        // for branches (if/else)
    condition: mongoose.Schema.Types.Mixed,
    nodeId: String
  }]
}, { timestamps: true });

WorkflowNodeSchema.index({ workflow: 1, nodeId: 1 }, { unique: true });
module.exports = mongoose.model('WorkflowNode', WorkflowNodeSchema);
```

```javascript
// models/WorkflowExecution.js  ← Tracks every contact's journey
const WorkflowExecutionSchema = new mongoose.Schema({
  workflow: { type: mongoose.Schema.Types.ObjectId, ref: 'Workflow', required: true },
  contact: { type: mongoose.Schema.Types.ObjectId, ref: 'Contact', required: true }, // or Lead

  currentNodeId: String,
  status: {
    type: String,
    enum: ['running', 'waiting', 'completed', 'failed', 'exited'],
    default: 'running'
  },

  history: [{
    nodeId: String,
    action: String,        // "email_sent", "delay_started", etc.
    result: mongoose.Schema.Types.Mixed,
    executedAt: { type: Date, default: Date.now }
  }],

  enrolledAt: { type: Date, default: Date.now },
  completedAt: Date,
  exitedAt: Date
}, { timestamps: true });

WorkflowExecutionSchema.index({ contact: 1, workflow: 1 });
WorkflowExecutionSchema.index({ status: 1, currentNodeId: 1 });

module.exports = mongoose.model('WorkflowExecution', WorkflowExecutionSchema);
```

### 2. Recommended Additional Fields in Campaign Model (add this)

```javascript
// Add this inside your Campaign schema
workflowAutomation: {
  enabled: { type: Boolean, default: false },
  entryWorkflow: { type: mongoose.Schema.Types.ObjectId, ref: 'Workflow' },     // runs when contact is added
  exitWorkflow: { type: mongoose.Schema.Types.ObjectId, ref: 'Workflow' },      // runs on removal/completion
  nurturingWorkflow: { type: mongoose.Schema.Types.ObjectId, ref: 'Workflow' }  // long-term drip
}
```

### 3. How the Whole Process Works (Real-World Flow)

```
Contact Added to Campaign
        ↓
Check campaign.workflowAutomation.entryWorkflow
        ↓
Create WorkflowExecution (status: running)
        ↓
Start executing from trigger node → first action node
        ↓
Actions can be:
   → Send Email → queue in BullMQ/Redis
   → Delay 3 days → store next run time
   → If/Else → evaluate conditions → branch
   → Goal reached → mark execution completed + update campaign metrics
```

### 4. Background Job Processor (Critical for Reliability)

Use **BullMQ + Redis** (best in production 2025)

```javascript
// workers/workflowWorker.js
import { Queue, Worker, QueueScheduler } from 'bullmq';
import IORedis from 'ioredis';

const connection = new IORedis(process.env.REDIS_URL);
const workflowQueue = new Queue('workflow-actions', { connection });
new QueueScheduler('workflow-actions', { connection });

const worker = new Worker('workflow-actions', async (job) => {
  const { executionId, nodeId } = job.data;

  const execution = await WorkflowExecution.findById(executionId)
    .populate('workflow contact');

  const node = await WorkflowNode.findOne({ workflow: execution.workflow, nodeId });

  switch (node.type) {
    case 'send_email':
      await sendEmailViaResendOrSES(node.config, execution.contact);
      await logHistory(execution, nodeId, 'email_sent');
      break;

    case 'delay':
      const delayMs = calcDelayMs(node.config);
      await workflowQueue.add('continue-workflow', { executionId, nextNodeId: node.nextNodeId }, { delay: delayMs });
      await execution.updateOne({ status: 'waiting' });
      return; // important – don't continue immediately

    case 'add_tag':
      await Contact.updateOne({ _id: execution.contact._id }, { $addToSet: { tags: node.config.tagId } });
      break;

    case 'if_else':
      const branch = evaluateConditions(node.config.conditions, execution.contact);
      const nextNodeId = branch ? branch.trueNodeId : branch.falseNodeId;
      await workflowQueue.add('continue-workflow', { executionId, nextNodeId });
      return;
  }

  // Continue to next node immediately (unless delayed)
  const nextNodeId = node.nextNodeId || node.nextNodes[0]?.nodeId;
  if (nextNodeId) {
    await workflowQueue.add('continue-workflow', { executionId, nextNodeId: nextNodeId }, { delay: 1000 });
  } else {
    await execution.updateOne({ status: 'completed', completedAt: new Date() });
  }
}, { connection, concurrency: 10 });
```

### 5. API Routes You Must Have

```javascript
POST   /api/workflows                    → create workflow (canvas JSON)
GET    /api/workflows/:id                → get full workflow + nodes
POST   /api/workflows/:id/enroll/:contactId
POST   /api/workflows/:id/trigger-manual/:contactId
DELETE /api/executions/:id               → cancel workflow for contact
GET    /api/campaigns/:id/analytics/workflows → dashboard stats
```

### 6. Frontend Builder Recommendation

Use **React Flow** (best in class)

```bash
npm install reactflow
```

You’ll save canvasData as JSON with nodes/edges → perfect round-trip.

### Final Architecture Summary (Scalable & Professional)

```
Redis + BullMQ          ← All async actions (emails, delays)
MongoDB                 ← Workflow, Node, Execution models
Express API             ← CRUD + enrollment endpoints
React + React Flow      ← Beautiful visual workflow builder
Webhook & Event System  ← Trigger workflows from forms, tags, etc.
```

This exact architecture is used by modern SaaS CRMs making $10M+ ARR.

Implement this and you’ll have **true marketing automation** — not just campaigns.

Would you like me to generate:
- Complete React Flow workflow builder component
- Full BullMQ worker with email + SMS + tag actions
- MongoDB aggregation queries for workflow analytics dashboard

? Say the word and I’ll give you production code.