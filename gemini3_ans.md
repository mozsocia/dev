This is a significant advanced feature for a CRM. To add **Workflow Automation** to your MERN project in a scalable, professional way, you should **not** stuff the logic inside the `Campaign` model.

Instead, you should create a separate **`Workflow`** entity that references the Campaign. This adheres to the **Single Responsibility Principle**: The Campaign model holds strategy and data; the Workflow model holds logic and execution rules.

Here is the professional implementation guide.

---

### 1. The Architecture: How it Works
To make this scalable, we don't run workflows directly in the API response (which would slow down the user experience). We use an **Event-Driven Architecture** with a **Job Queue**.

1.  **Trigger:** Something happens (e.g., a Lead enters the Campaign).
2.  **Event:** The system emits an event.
3.  **Listener:** A background service looks for active Workflows associated with that Trigger and Campaign.
4.  **Queue:** The workflow steps are pushed to a queue (e.g., Redis/BullMQ).
5.  **Worker:** A background worker executes the steps (Send Email -> Wait 2 Days -> Check Status).

---

### 2. Database Schema Updates

Do not modify the `Campaign` schema heavily. Instead, create a new model.

#### New Model: `Workflow.js`
This model defines the rules (the "Flowchart").

```javascript
// models/Workflow.js
const mongoose = require('mongoose');

const stepSchema = new mongoose.Schema({
  stepId: { type: String, required: true }, // uuid
  type: { 
    type: String, 
    enum: ['action', 'delay', 'condition'], 
    required: true 
  },
  // Specific configuration based on type
  actionType: { 
    type: String, 
    enum: ['send_email', 'update_lead_score', 'create_task', 'send_sms', 'notify_user'] 
  },
  // Flexible config object
  config: {
    templateId: { type: mongoose.Schema.Types.ObjectId, ref: 'EmailTemplate' }, // If email
    delayDuration: Number, // In minutes, if delay
    delayUnit: { type: String, enum: ['minutes', 'hours', 'days'] },
    conditionField: String, // e.g., "lead.score"
    conditionOperator: String, // e.g., "greater_than"
    conditionValue: mongoose.Schema.Types.Mixed
  },
  nextStepId: String, // Pointer to the next step (Linked List style)
  // For branching logic (Conditions)
  trueNextStepId: String, 
  falseNextStepId: String
});

const workflowSchema = new mongoose.Schema({
  name: { type: String, required: true },
  campaign: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Campaign',
    required: true
  },
  isActive: { type: Boolean, default: false },
  
  // What starts this workflow?
  trigger: {
    type: {
      type: String,
      enum: ['lead_added', 'lead_status_changed', 'form_submitted', 'tag_added', 'manual']
    },
    config: mongoose.Schema.Types.Mixed
  },
  
  // The flowchart nodes
  steps: [stepSchema],
  
  // Stats
  stats: {
    enrolled: { type: Number, default: 0 },
    completed: { type: Number, default: 0 },
    active: { type: Number, default: 0 }
  }
}, { timestamps: true });

module.exports = mongoose.model('Workflow', workflowSchema);
```

#### New Model: `WorkflowExecution.js`
This model tracks a specific Lead going through a specific Workflow (The "Instance").

```javascript
// models/WorkflowExecution.js
const mongoose = require('mongoose');

const executionSchema = new mongoose.Schema({
  workflow: { type: mongoose.Schema.Types.ObjectId, ref: 'Workflow' },
  campaign: { type: mongoose.Schema.Types.ObjectId, ref: 'Campaign' },
  lead: { type: mongoose.Schema.Types.ObjectId, ref: 'Lead' }, // Assuming you have a Lead model
  
  currentStepId: String,
  status: { 
    type: String, 
    enum: ['pending', 'active', 'waiting', 'completed', 'failed', 'cancelled'],
    default: 'pending'
  },
  
  history: [{
    stepId: String,
    actionType: String,
    status: String, // success, failed
    executedAt: Date,
    error: String,
    metadata: mongoose.Schema.Types.Mixed // e.g., emailId sent
  }],
  
  scheduledResumeTime: Date // Used if the status is 'waiting' (Delay step)
}, { timestamps: true });

module.exports = mongoose.model('WorkflowExecution', executionSchema);
```

---

### 3. Implementation Guide (Backend)

To make this scalable, you should use **BullMQ (with Redis)** for queue management. This handles the "Wait 2 days" logic without freezing your server.

**Prerequisites:**
`npm install bullmq ioredis`

#### Step A: The Workflow Engine (Core Logic)

Create a service that handles moving a lead to the next step.

```javascript
// services/workflowEngine.js
const Workflow = require('../models/Workflow');
const WorkflowExecution = require('../models/WorkflowExecution');
const { workflowQueue } = require('../queues/bullConfig'); // See Step B

const startWorkflow = async (workflowId, leadId) => {
  const workflow = await Workflow.findById(workflowId);
  if (!workflow || !workflow.isActive) return;

  // Create Execution Record
  const execution = await WorkflowExecution.create({
    workflow: workflowId,
    campaign: workflow.campaign,
    lead: leadId,
    currentStepId: workflow.steps[0].stepId, // Start at step 1
    status: 'active'
  });

  // Add to Queue
  await workflowQueue.add('processStep', { 
    executionId: execution._id 
  });
};

const processStep = async (executionId) => {
  const execution = await WorkflowExecution.findById(executionId)
    .populate('workflow')
    .populate('lead');

  if (!execution || execution.status !== 'active') return;

  const currentStep = execution.workflow.steps.find(s => s.stepId === execution.currentStepId);
  
  if (!currentStep) {
    execution.status = 'completed';
    await execution.save();
    return;
  }

  try {
    // --- HANDLE ACTIONS ---
    if (currentStep.type === 'action') {
      await executeAction(currentStep, execution.lead);
      // Move to next
      await advanceWorkflow(execution, currentStep.nextStepId);
    } 
    
    // --- HANDLE DELAYS ---
    else if (currentStep.type === 'delay') {
      const delayMs = calculateDelay(currentStep.config);
      execution.status = 'waiting';
      execution.scheduledResumeTime = new Date(Date.now() + delayMs);
      await execution.save();

      // Add back to queue with delay
      await workflowQueue.add('processStep', { 
        executionId: execution._id 
      }, { delay: delayMs });
    }

    // --- HANDLE CONDITIONS ---
    else if (currentStep.type === 'condition') {
      const isTrue = evaluateCondition(currentStep.config, execution.lead);
      const nextId = isTrue ? currentStep.trueNextStepId : currentStep.falseNextStepId;
      await advanceWorkflow(execution, nextId);
    }

  } catch (error) {
    execution.status = 'failed';
    execution.history.push({ stepId: currentStep.stepId, error: error.message, executedAt: new Date() });
    await execution.save();
  }
};

// Helper to move pointer
const advanceWorkflow = async (execution, nextStepId) => {
  execution.history.push({ 
    stepId: execution.currentStepId, 
    status: 'success', 
    executedAt: new Date() 
  });
  
  if (!nextStepId) {
    execution.status = 'completed';
  } else {
    execution.currentStepId = nextStepId;
    // Re-queue immediately for next step
    await workflowQueue.add('processStep', { executionId: execution._id });
  }
  await execution.save();
};

module.exports = { startWorkflow, processStep };
```

#### Step B: The Queue Worker (Background Processor)

This file runs separately (or alongside your server) to process the jobs.

```javascript
// workers/workflowWorker.js
const { Worker } = require('bullmq');
const { processStep } = require('../services/workflowEngine');
const connection = { host: 'localhost', port: 6379 }; // Redis config

const worker = new Worker('WorkflowQueue', async (job) => {
  console.log(`Processing step for Execution: ${job.data.executionId}`);
  await processStep(job.data.executionId);
}, { connection });

worker.on('completed', job => {
  console.log(`Job ${job.id} has completed!`);
});

worker.on('failed', (job, err) => {
  console.log(`Job ${job.id} has failed with ${err.message}`);
});
```

#### Step C: Hooking it into your Controllers

When a user performs an action (like adding a lead), check for workflows.

```javascript
// controllers/leadController.js
const Lead = require('../models/Lead');
const Workflow = require('../models/Workflow');
const { startWorkflow } = require('../services/workflowEngine');

exports.createLead = async (req, res) => {
  // 1. Create Lead
  const lead = await Lead.create(req.body);
  
  // 2. Find Automation (Trigger: lead_added)
  const workflows = await Workflow.find({
    campaign: req.body.campaignId,
    isActive: true,
    'trigger.type': 'lead_added'
  });

  // 3. Trigger Workflows
  workflows.forEach(wf => {
    startWorkflow(wf._id, lead._id);
  });

  res.status(201).json({ success: true, data: lead });
};
```

---

### 4. Scalability & Best Practices

1.  **Redis is Mandatory:** Do not use `setTimeout` in Node.js for delays (e.g., "Wait 2 days"). If the server restarts, you lose the timer. Redis/BullMQ persists the job, so if your server reboots, the email still gets sent 2 days later.
2.  **Idempotency:** Ensure that if a step runs twice by accident, it doesn't ruin data (e.g., check `history` before sending an email to ensure it wasn't already sent).
3.  **Error Handling:** If an external API (like SendGrid) is down, your workflow will fail. BullMQ supports `attempts` and `backoff`. Configure retries automatically.
4.  **Atomic Transactions:** When updating the Campaign stats and Workflow stats, use MongoDB Transactions (`session`) to ensure numbers don't get out of sync.

### 5. How to Integrate with Campaign Model

In your `Campaign.js`, you don't need to add heavy fields, but you can add a **Virtual Populate** so you can easily see workflows when querying a campaign.

```javascript
// In models/Campaign.js

// Add this before model compilation
campaignSchema.virtual('workflows', {
  ref: 'Workflow',
  localField: '_id',
  foreignField: 'campaign'
});

// Allow virtuals
campaignSchema.set('toObject', { virtuals: true });
campaignSchema.set('toJSON', { virtuals: true });
```

### Summary of Workflow Process

1.  **User creates a Campaign** (e.g., "Summer Sale").
2.  **User creates a Workflow** via UI (Visual builder).
    *   *Step 1:* Trigger: Lead Added.
    *   *Step 2:* Action: Send Welcome Email.
    *   *Step 3:* Delay: Wait 1 Day.
    *   *Step 4:* Action: Send Follow-up Email.
3.  **API saves** this to `Workflow` collection.
4.  **New Lead is added** to "Summer Sale".
5.  **Controller** finds the workflow and calls `startWorkflow`.
6.  **BullMQ** takes over, processing steps one by one, handling the delays and failures automatically.