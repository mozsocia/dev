i have crm project in MERN where i need to use below campaign model

now i need to add workflow automation , how to add workclow automation ? the fields needed? implement guide and how the whole process works 

do the implementation in professional scallable and best standard way

# CRM project
#### 16. **Campaign Model**

```javascript
// models/Campaign.js
const mongoose = require('mongoose');

const campaignSchema = new mongoose.Schema({
  // Basic Information
  name: {
    type: String,
    required: [true, 'Campaign name is required'],
    trim: true,
    maxlength: 200
  },
  code: {
    type: String,
    unique: true,
    uppercase: true,
    trim: true
  },
  description: String,

  // Type & Objective
  type: {
    type: String,
    enum: [
      'email',
      'social_media',
      'ppc',
      'webinar',
      'event',
      'content_marketing',
      'referral',
      'partner',
      'cold_outreach',
      'retargeting',
      'nurturing',
      'other'
    ],
    required: true
  },
  objective: {
    type: String,
    enum: ['lead_generation', 'brand_awareness', 'engagement', 'sales', 'retention', 'upsell', 'winback'],
    required: true
  },

  channels: [{
    name: {
      type: String,
      enum: ['email', 'facebook', 'instagram', 'linkedin', 'twitter', 
             'google_ads', 'youtube', 'website', 'sms', 'direct_mail', 'other']
    },
    budget: Number,
    spend: Number,
    isActive: {
      type: Boolean,
      default: true
    }
  }],

  // Status & Timeline
  startDate: {
    type: Date,
    required: true
  },
  endDate: {
    type: Date,
    required: true
  },
  actualStartDate: Date,
  actualEndDate: Date,

  status: {
    type: String,
    enum: ['planning', 'active', 'paused', 'completed', 'cancelled'],
    default: 'planning'
  },

  
  // Budget & Cost
  budget: {
    planned: {
      type: Number,
      required: true
    },
    actual: {
      type: Number,
      default: 0
    },
    currency: {
      type: String,
      default: 'USD'
    },
    costPerLead: Number,
    costPerConversion: Number,
    roi: Number
  },

  cost: {
    advertising: { type: Number, default: 0 },
    production: { type: Number, default: 0 },
    personnel: { type: Number, default: 0 },
    other: { type: Number, default: 0 },
    costPerLead: Number,
    costPerConversion: Number,
    roi: Number
  },

    // Targeting & Audience
  targetAudience: {
    industries: [String],
    companySizes: [String],
    jobTitles: [String],
    locations: [String],
    tags: [{
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Tag'
    }],
    segments: [String],
    totalAudienceSize: Number,
    estimatedReach: Number,
    actualReach: Number
  },

    // Campaign Content
  content: {
    emailTemplate: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'EmailTemplate'
    },
    landingPageUrl: String,
    creativeAssets: [{
      type: {
        type: String,
        enum: ['image', 'video', 'document', 'banner', 'other']
      },
      name: String,
      url: String,
      size: Number,
      dimensions: String
    }],
    messaging: {
      headline: String,
      subheadline: String,
      cta: String,
      body: String
    }
  },

    // Goals and KPIs
  goals: {
    // Existing
    targetLeads: Number,
    targetConversions: Number,
    targetRevenue: Number,
    targetEngagement: Number,

    // ← NEW → recommended additions
    targetImpressions: Number,
    targetReach: Number,
    targetClicks: Number,
    targetCTR: Number,               // e.g. 2.5 for 2.5%
    targetConversionRate: Number,    // e.g. 5.0 for 5%
    targetCostPerLead: Number,
    targetCostPerConversion: Number,
    targetROI: Number,               // e.g. 300 for 300% ROI
    targetOpenRate: Number,          // optional, only used for email campaigns
    targetBounceRate: Number,        // optional
    // keep customGoals array for everything else
    customGoals: [{ name: String, target: Number, actual: Number }]
  }

   
  // Performance Metrics
  metrics: {
    // Reach Metrics
    impressions: { type: Number, default: 0 },
    reach: { type: Number, default: 0 },
    
    // Engagement Metrics
    clicks: { type: Number, default: 0 },
    ctr: { type: Number, default: 0 },
    engagement: { type: Number, default: 0 },
    
    // Conversion Metrics
    leads: { type: Number, default: 0 },
    conversions: { type: Number, default: 0 },
    conversionRate: { type: Number, default: 0 },
    
    // Email Specific
    emailsSent: { type: Number, default: 0 },
    emailsOpened: { type: Number, default: 0 },
    emailsClicked: { type: Number, default: 0 },
    emailsBounced: { type: Number, default: 0 },
    unsubscribed: { type: Number, default: 0 },
    
    // Revenue Metrics
    revenue: { type: Number, default: 0 },
    averageDealSize: Number,
    customerAcquisitionCost: Number
  },
  
  // Reporting
  reports: [{
    type: {
      type: String,
      enum: ['daily', 'weekly', 'monthly', 'custom']
    },
    generatedAt: Date,
    url: String
  }],

  // Notes and Attachments
  notes: String,
  attachments: [{
    name: String,
    url: String,
    type: String,
    size: Number,
    uploadedBy: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    },
    uploadedAt: Date
  }],
  
  // Custom Fields
  customFields: mongoose.Schema.Types.Mixed,


  // Metadata
  lastActivityDate: Date,
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  updatedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  approvedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  approvedAt: Date,
  deletedAt: Date
}, {
  timestamps: true,
  toJSON: { virtuals: true },
  toObject: { virtuals: true }
});
```