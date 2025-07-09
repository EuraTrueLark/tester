import React, { useState, useCallback, useEffect, useMemo, useReducer, createContext, useContext } from 'react';

// Test environment detection and polyfills
if (typeof window !== 'undefined' && !window.matchMedia) {
  window.matchMedia = (query) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: () => {},
    removeListener: () => {},
    addEventListener: () => {},
    removeEventListener: () => {},
    dispatchEvent: () => {},
  });
}

// ========================================
// CORE STATE MANAGEMENT & CONTEXT
// ========================================

// Authentication Context with proper error handling
const AuthContext = createContext(null);
const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};

// Organization Context with proper error handling
const OrganizationContext = createContext(null);
const useOrganization = () => {
  const context = useContext(OrganizationContext);
  if (!context) {
    throw new Error('useOrganization must be used within an OrganizationProvider');
  }
  return context;
};

// WebSocket Context for Real-time Updates with proper error handling
const WebSocketContext = createContext(null);
const useWebSocket = () => {
  const context = useContext(WebSocketContext);
  if (!context) {
    throw new Error('useWebSocket must be used within a WebSocketProvider');
  }
  return context;
};

// ========================================
// UTILITY FUNCTIONS & HELPERS
// ========================================

const generateId = () => Math.random().toString(36).substr(2, 9);

const formatDate = (date) => {
  return new Intl.DateTimeFormat('en-US', { 
    dateStyle: 'medium', 
    timeStyle: 'short' 
  }).format(new Date(date));
};

const calculateHealthScore = (customer) => {
  // Advanced health scoring algorithm based on multiple factors
  const factors = {
    engagement: customer.lastActivity ? Math.max(0, 100 - (Date.now() - new Date(customer.lastActivity).getTime()) / (1000 * 60 * 60 * 24) * 2) : 0,
    usage: customer.productUsage || 50,
    satisfaction: customer.npsScore || 50,
    support: Math.max(0, 100 - (customer.supportTickets || 0) * 10),
    revenue: customer.mrr ? Math.min(100, customer.mrr / 100) : 50
  };
  
  const weights = { engagement: 0.3, usage: 0.25, satisfaction: 0.2, support: 0.15, revenue: 0.1 };
  
  return Math.round(
    Object.entries(factors).reduce((score, [key, value]) => score + value * weights[key], 0)
  );
};

// ========================================
// MOCK DATA & SERVICES
// ========================================

const mockUsers = [
  { 
    id: '1', 
    email: 'admin@acme.com', 
    name: 'John Admin', 
    role: 'ADMIN', 
    organizationId: 'org1',
    mfaEnabled: true,
    lastLogin: new Date().toISOString(),
    avatar: '👨‍💼'
  },
  { 
    id: '2', 
    email: 'manager@acme.com', 
    name: 'Sarah Manager', 
    role: 'MANAGER', 
    organizationId: 'org1',
    mfaEnabled: false,
    lastLogin: new Date(Date.now() - 86400000).toISOString(),
    avatar: '👩‍💼'
  }
];

const mockOrganizations = [
  {
    id: 'org1',
    name: 'Acme Corporation',
    slug: 'acme-corp',
    subscriptionTier: 'ENTERPRISE',
    settings: {
      customBranding: { primaryColor: '#1976d2', logo: '🏢' },
      integrations: ['salesforce', 'hubspot', 'slack'],
      features: ['ai_nodes', 'advanced_analytics', 'white_label']
    },
    createdAt: new Date('2024-01-01').toISOString(),
    userCount: 147,
    customerCount: 1247
  }
];

const mockCustomers = [
  { 
    id: '1', 
    name: 'TechStart Inc', 
    email: 'contact@techstart.com',
    healthScore: 45, 
    status: 'At Risk', 
    lastActivity: '2 days ago',
    risk: 'High',
    organizationId: 'org1',
    customAttributes: {
      industry: 'Technology',
      employees: 50,
      mrr: 2500,
      contractValue: 30000,
      renewalDate: '2024-12-15'
    },
    productUsage: 30,
    npsScore: 3,
    supportTickets: 8,
    tags: ['enterprise', 'at-risk', 'high-value'],
    assignedCSM: 'Sarah Manager',
    lifecycle: 'GROWTH',
    churnProbability: 0.75
  },
  { 
    id: '2', 
    name: 'Global Solutions', 
    email: 'team@globalsolutions.com',
    healthScore: 72, 
    status: 'Healthy', 
    lastActivity: '1 hour ago',
    risk: 'Medium',
    organizationId: 'org1',
    customAttributes: {
      industry: 'Consulting',
      employees: 200,
      mrr: 5000,
      contractValue: 60000,
      renewalDate: '2024-08-30'
    },
    productUsage: 75,
    npsScore: 8,
    supportTickets: 2,
    tags: ['enterprise', 'growing', 'advocate'],
    assignedCSM: 'John Admin',
    lifecycle: 'EXPANSION',
    churnProbability: 0.15
  },
  { 
    id: '3', 
    name: 'Innovation Labs', 
    email: 'hello@innovationlabs.com',
    healthScore: 92, 
    status: 'Excellent', 
    lastActivity: '30 minutes ago',
    risk: 'Low',
    organizationId: 'org1',
    customAttributes: {
      industry: 'Research',
      employees: 25,
      mrr: 1200,
      contractValue: 15000,
      renewalDate: '2025-03-01'
    },
    productUsage: 95,
    npsScore: 10,
    supportTickets: 0,
    tags: ['startup', 'advocate', 'high-usage'],
    assignedCSM: 'Sarah Manager',
    lifecycle: 'ADVOCACY',
    churnProbability: 0.02
  },
  { 
    id: '4', 
    name: 'Enterprise Corp', 
    email: 'procurement@enterprise.com',
    healthScore: 85, 
    status: 'Healthy', 
    lastActivity: '2 hours ago',
    risk: 'Low',
    organizationId: 'org1',
    customAttributes: {
      industry: 'Manufacturing',
      employees: 5000,
      mrr: 15000,
      contractValue: 180000,
      renewalDate: '2024-06-15'
    },
    productUsage: 80,
    npsScore: 9,
    supportTickets: 1,
    tags: ['enterprise', 'stable', 'strategic'],
    assignedCSM: 'John Admin',
    lifecycle: 'RETENTION',
    churnProbability: 0.05
  }
];

const mockPlaybooks = [
  { 
    id: '1', 
    name: 'Customer Onboarding Journey', 
    category: 'Onboarding', 
    triggers: 3, 
    isActive: true, 
    lastRun: '2 hours ago',
    organizationId: 'org1',
    version: '1.2.0',
    createdBy: 'John Admin',
    description: 'Complete customer onboarding workflow with welcome emails, training setup, and success check-ins',
    nodes: 12,
    avgExecutionTime: '45 minutes',
    successRate: 94.5,
    totalExecutions: 1247
  },
  { 
    id: '2', 
    name: 'Health Score Alert System', 
    category: 'Monitoring', 
    triggers: 8, 
    isActive: true, 
    lastRun: '15 minutes ago',
    organizationId: 'org1',
    version: '2.1.0',
    createdBy: 'Sarah Manager',
    description: 'Automated monitoring and intervention for declining customer health scores',
    nodes: 8,
    avgExecutionTime: '5 minutes',
    successRate: 98.2,
    totalExecutions: 2847
  },
  { 
    id: '3', 
    name: 'Renewal Campaign Automation', 
    category: 'Retention', 
    triggers: 2, 
    isActive: false, 
    lastRun: '1 week ago',
    organizationId: 'org1',
    version: '1.0.3',
    createdBy: 'John Admin',
    description: 'Automated renewal reminders and upsell opportunities 90 days before contract expiry',
    nodes: 15,
    avgExecutionTime: '2 hours',
    successRate: 87.3,
    totalExecutions: 156
  },
  { 
    id: '4', 
    name: 'Feature Adoption Accelerator', 
    category: 'Growth', 
    triggers: 5, 
    isActive: true, 
    lastRun: '6 hours ago',
    organizationId: 'org1',
    version: '1.1.2',
    createdBy: 'Sarah Manager',
    description: 'Identify low-usage customers and guide them through feature adoption workflows',
    nodes: 10,
    avgExecutionTime: '30 minutes',
    successRate: 76.8,
    totalExecutions: 567
  }
];

const mockExecutions = [
  { 
    id: '1', 
    playbookId: '1',
    playbookName: 'Customer Onboarding Journey', 
    status: 'Running', 
    progress: 65, 
    customer: 'TechStart Inc',
    customerId: '1',
    startTime: new Date(Date.now() - 7200000).toISOString(),
    currentNode: 'Email Welcome Sequence',
    executedBy: 'John Admin',
    triggerType: 'Manual',
    estimatedCompletion: new Date(Date.now() + 1800000).toISOString()
  },
  { 
    id: '2', 
    playbookId: '2',
    playbookName: 'Health Score Alert System', 
    status: 'Completed', 
    progress: 100, 
    customer: 'Global Solutions',
    customerId: '2',
    startTime: new Date(Date.now() - 86400000).toISOString(),
    currentNode: 'Completed',
    executedBy: 'System',
    triggerType: 'Automated',
    estimatedCompletion: new Date(Date.now() - 86100000).toISOString()
  },
  { 
    id: '3', 
    playbookId: '4',
    playbookName: 'Feature Adoption Accelerator', 
    status: 'Paused', 
    progress: 30, 
    customer: 'Innovation Labs',
    customerId: '3',
    startTime: new Date(Date.now() - 10800000).toISOString(),
    currentNode: 'Feature Usage Analysis',
    executedBy: 'Sarah Manager',
    triggerType: 'Scheduled',
    estimatedCompletion: null
  },
  { 
    id: '4', 
    playbookId: '1',
    playbookName: 'Customer Onboarding Journey', 
    status: 'Failed', 
    progress: 45, 
    customer: 'Enterprise Corp',
    customerId: '4',
    startTime: new Date(Date.now() - 18000000).toISOString(),
    currentNode: 'CRM Integration',
    executedBy: 'System',
    triggerType: 'Webhook',
    estimatedCompletion: null,
    errorMessage: 'Failed to connect to Salesforce API - Rate limit exceeded'
  }
];

// Enhanced Node Types with AI/ML capabilities
const NodeTypes = {
  // Trigger Nodes
  manual: { label: 'Manual Trigger', icon: '👤', color: 'bg-blue-500', category: 'triggers', description: 'Manual playbook execution' },
  webhook: { label: 'Webhook Trigger', icon: '🔗', color: 'bg-green-500', category: 'triggers', description: 'External API webhook trigger' },
  scheduled: { label: 'Scheduled Trigger', icon: '⏰', color: 'bg-purple-500', category: 'triggers', description: 'Time-based cron trigger' },
  event: { label: 'Event Trigger', icon: '⚡', color: 'bg-yellow-500', category: 'triggers', description: 'Customer event-based trigger' },
  
  // Communication Nodes
  email: { label: 'Email', icon: '📧', color: 'bg-blue-600', category: 'communication', description: 'Send personalized emails' },
  sms: { label: 'SMS', icon: '💬', color: 'bg-green-600', category: 'communication', description: 'Send SMS messages' },
  slack: { label: 'Slack Message', icon: '💬', color: 'bg-purple-600', category: 'communication', description: 'Send Slack notifications' },
  inapp: { label: 'In-App Message', icon: '📱', color: 'bg-indigo-600', category: 'communication', description: 'Send in-app notifications' },
  
  // Logic Nodes
  condition: { label: 'Condition', icon: '❓', color: 'bg-orange-500', category: 'logic', description: 'Conditional branching logic' },
  loop: { label: 'Loop', icon: '🔄', color: 'bg-cyan-500', category: 'logic', description: 'Iterate over data sets' },
  wait: { label: 'Wait', icon: '⏱️', color: 'bg-gray-500', category: 'logic', description: 'Delay execution' },
  parallel: { label: 'Parallel', icon: '🔀', color: 'bg-teal-500', category: 'logic', description: 'Execute multiple branches' },
  
  // Data Nodes
  database: { label: 'Database', icon: '🗄️', color: 'bg-red-500', category: 'data', description: 'Database operations' },
  api: { label: 'HTTP Request', icon: '🌐', color: 'bg-blue-500', category: 'data', description: 'External API calls' },
  transform: { label: 'Transform', icon: '🔄', color: 'bg-yellow-600', category: 'data', description: 'Data transformation' },
  filter: { label: 'Filter', icon: '🔍', color: 'bg-pink-500', category: 'data', description: 'Filter data sets' },
  
  // AI/ML Nodes
  sentiment: { label: 'Sentiment Analysis', icon: '😊', color: 'bg-purple-700', category: 'ai', description: 'Analyze text sentiment' },
  prediction: { label: 'Churn Prediction', icon: '🔮', color: 'bg-red-700', category: 'ai', description: 'ML-powered predictions' },
  classification: { label: 'Text Classification', icon: '📊', color: 'bg-green-700', category: 'ai', description: 'Classify text content' },
  llm: { label: 'LLM Processor', icon: '🧠', color: 'bg-violet-700', category: 'ai', description: 'Large Language Model integration' },
  
  // Integration Nodes
  salesforce: { label: 'Salesforce', icon: '☁️', color: 'bg-blue-400', category: 'integrations', description: 'Salesforce CRM integration' },
  hubspot: { label: 'HubSpot', icon: '🎯', color: 'bg-orange-400', category: 'integrations', description: 'HubSpot CRM integration' },
  zendesk: { label: 'Zendesk', icon: '🎫', color: 'bg-green-400', category: 'integrations', description: 'Zendesk support integration' },
  stripe: { label: 'Stripe', icon: '💳', color: 'bg-purple-400', category: 'integrations', description: 'Stripe payment integration' }
};

// ========================================
// AUTHENTICATION & AUTHORIZATION
// ========================================

const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(mockUsers[0]); // Start with admin user
  const [isAuthenticated, setIsAuthenticated] = useState(true);
  const [mfaRequired, setMfaRequired] = useState(false);

  const login = async (email, password, mfaCode = null) => {
    // Simulate authentication
    const foundUser = mockUsers.find(u => u.email === email);
    if (foundUser) {
      if (foundUser.mfaEnabled && !mfaCode) {
        setMfaRequired(true);
        return { requiresMFA: true };
      }
      setUser(foundUser);
      setIsAuthenticated(true);
      setMfaRequired(false);
      return { success: true, user: foundUser };
    }
    return { error: 'Invalid credentials' };
  };

  const logout = () => {
    setUser(null);
    setIsAuthenticated(false);
    setMfaRequired(false);
  };

  const hasPermission = (permission) => {
    if (!user) return false;
    
    const rolePermissions = {
      SUPER_ADMIN: ['*'],
      ADMIN: ['manage_org', 'manage_users', 'manage_playbooks', 'view_analytics', 'execute_playbooks'],
      MANAGER: ['manage_playbooks', 'view_analytics', 'execute_playbooks', 'manage_team'],
      AGENT: ['execute_playbooks', 'view_customers', 'view_executions'],
      VIEWER: ['view_customers', 'view_executions']
    };

    const userPermissions = rolePermissions[user.role] || [];
    return userPermissions.includes('*') || userPermissions.includes(permission);
  };

  return (
    <AuthContext.Provider value={{
      user,
      isAuthenticated,
      mfaRequired,
      login,
      logout,
      hasPermission
    }}>
      {children}
    </AuthContext.Provider>
  );
};

// ========================================
// ORGANIZATION MANAGEMENT
// ========================================

const OrganizationProvider = ({ children }) => {
  const [currentOrg, setCurrentOrg] = useState(mockOrganizations[0]);
  const [organizations, setOrganizations] = useState(mockOrganizations);

  const switchOrganization = (orgId) => {
    const org = organizations.find(o => o.id === orgId);
    if (org) {
      setCurrentOrg(org);
    }
  };

  const updateOrgSettings = (settings) => {
    setCurrentOrg(prev => ({ ...prev, settings: { ...prev.settings, ...settings } }));
  };

  return (
    <OrganizationContext.Provider value={{
      currentOrg,
      organizations,
      switchOrganization,
      updateOrgSettings
    }}>
      {children}
    </OrganizationContext.Provider>
  );
};

// ========================================
// WEBSOCKET & REAL-TIME UPDATES
// ========================================

const WebSocketProvider = ({ children }) => {
  const [connected, setConnected] = useState(false);
  const [notifications, setNotifications] = useState([]);
  const [realtimeData, setRealtimeData] = useState({});
  const [connectionError, setConnectionError] = useState(null);

  useEffect(() => {
    let interval;
    let connectionTimeout;
    
    try {
      // Simulate WebSocket connection with proper error handling
      connectionTimeout = setTimeout(() => {
        setConnected(true);
        setConnectionError(null);
      }, 100);
      
      interval = setInterval(() => {
        if (!connected) return;
        
        try {
          // Simulate real-time updates with error handling
          const updates = [
            { type: 'execution_update', data: { executionId: '1', progress: Math.min(100, Math.random() * 100) } },
            { type: 'health_score_change', data: { customerId: '1', newScore: Math.floor(Math.random() * 100) } },
            { type: 'new_customer_signup', data: { customerName: 'New Customer Corp' } },
            { type: 'playbook_completed', data: { playbookName: 'Onboarding Journey', customer: 'Acme Corp' } }
          ];
          
          const randomUpdate = updates[Math.floor(Math.random() * updates.length)];
          
          setNotifications(prev => [...prev, {
            id: generateId(),
            message: getNotificationMessage(randomUpdate),
            type: randomUpdate.type,
            timestamp: new Date().toISOString(),
            read: false
          }].slice(-10)); // Keep only last 10 notifications
          
          setRealtimeData(prev => ({ ...prev, [randomUpdate.type]: randomUpdate.data }));
        } catch (error) {
          console.error('Real-time update error:', error);
          setConnectionError(error.message);
        }
      }, 5000);
    } catch (error) {
      console.error('WebSocket connection error:', error);
      setConnectionError(error.message);
      setConnected(false);
    }

    return () => {
      if (interval) clearInterval(interval);
      if (connectionTimeout) clearTimeout(connectionTimeout);
    };
  }, [connected]);

  const getNotificationMessage = (update) => {
    try {
      switch (update.type) {
        case 'execution_update':
          return `Workflow execution progress: ${Math.floor(update.data.progress)}%`;
        case 'health_score_change':
          return `Customer health score updated: ${update.data.newScore}`;
        case 'new_customer_signup':
          return `New customer signup: ${update.data.customerName}`;
        case 'playbook_completed':
          return `Playbook "${update.data.playbookName}" completed for ${update.data.customer}`;
        default:
          return 'System update received';
      }
    } catch (error) {
      console.error('Notification message error:', error);
      return 'System update received';
    }
  };

  const markNotificationRead = useCallback((notificationId) => {
    setNotifications(prev => 
      prev.map(n => n.id === notificationId ? { ...n, read: true } : n)
    );
  }, []);

  const clearNotifications = useCallback(() => {
    setNotifications([]);
  }, []);

  const reconnect = useCallback(() => {
    setConnected(false);
    setConnectionError(null);
    // Reconnection logic would go here
    setTimeout(() => setConnected(true), 1000);
  }, []);

  return (
    <WebSocketContext.Provider value={{
      connected,
      notifications,
      realtimeData,
      connectionError,
      markNotificationRead,
      clearNotifications,
      reconnect
    }}>
      {children}
    </WebSocketContext.Provider>
  );
};

// ========================================
// ENHANCED DASHBOARD COMPONENTS
// ========================================

const AdvancedHealthScoreCard = ({ customer }) => {
  const { currentOrg } = useOrganization();
  const [expanded, setExpanded] = useState(false);
  
  const healthScore = calculateHealthScore(customer);
  const trend = customer.healthTrend || (Math.random() > 0.5 ? 'up' : 'down');
  
  const getHealthColor = (score) => {
    if (score >= 80) return 'text-green-600';
    if (score >= 60) return 'text-yellow-600';
    return 'text-red-600';
  };

  const getHealthBg = (score) => {
    if (score >= 80) return 'bg-green-500';
    if (score >= 60) return 'bg-yellow-500';
    return 'bg-red-500';
  };

  const getRiskColor = (risk) => {
    switch (risk) {
      case 'Low': return 'bg-green-100 text-green-800';
      case 'Medium': return 'bg-yellow-100 text-yellow-800';
      case 'High': return 'bg-red-100 text-red-800';
      default: return 'bg-gray-100 text-gray-800';
    }
  };

  const getChurnRisk = () => {
    if (customer.churnProbability > 0.7) return 'High';
    if (customer.churnProbability > 0.3) return 'Medium';
    return 'Low';
  };

  return (
    <div className="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow">
      <div className="flex justify-between items-start mb-4">
        <div>
          <h3 className="text-lg font-semibold text-gray-800">{customer.name}</h3>
          <p className="text-sm text-gray-600">{customer.customAttributes?.industry}</p>
        </div>
        <div className="flex gap-2">
          <span className={`px-2 py-1 rounded-full text-xs font-medium ${getRiskColor(getChurnRisk())}`}>
            {getChurnRisk()} Risk
          </span>
          <span className="text-sm">
            {trend === 'up' ? '📈' : '📉'}
          </span>
        </div>
      </div>

      <div className="flex items-center gap-4 mb-4">
        <div className="flex-1">
          <div className="flex justify-between items-center mb-1">
            <span className="text-sm text-gray-600">Health Score</span>
            <span className={`text-lg font-bold ${getHealthColor(healthScore)}`}>
              {healthScore}
            </span>
          </div>
          <div className="w-full bg-gray-200 rounded-full h-3">
            <div 
              className={`h-3 rounded-full transition-all duration-500 ${getHealthBg(healthScore)}`}
              style={{ width: `${healthScore}%` }}
            ></div>
          </div>
        </div>
      </div>

      <div className="grid grid-cols-2 gap-4 mb-4">
        <div>
          <p className="text-xs text-gray-500">MRR</p>
          <p className="font-semibold">${customer.customAttributes?.mrr?.toLocaleString()}</p>
        </div>
        <div>
          <p className="text-xs text-gray-500">Contract Value</p>
          <p className="font-semibold">${customer.customAttributes?.contractValue?.toLocaleString()}</p>
        </div>
      </div>

      <div className="flex justify-between items-center">
        <p className="text-sm text-gray-600">
          CSM: {customer.assignedCSM}
        </p>
        <button 
          onClick={() => setExpanded(!expanded)}
          className="text-blue-600 text-sm hover:underline"
        >
          {expanded ? 'Less' : 'More'} Details
        </button>
      </div>

      {expanded && (
        <div className="mt-4 pt-4 border-t space-y-3">
          <div className="grid grid-cols-2 gap-4 text-sm">
            <div>
              <p className="text-gray-500">Product Usage</p>
              <p className="font-medium">{customer.productUsage}%</p>
            </div>
            <div>
              <p className="text-gray-500">NPS Score</p>
              <p className="font-medium">{customer.npsScore}/10</p>
            </div>
            <div>
              <p className="text-gray-500">Support Tickets</p>
              <p className="font-medium">{customer.supportTickets}</p>
            </div>
            <div>
              <p className="text-gray-500">Lifecycle</p>
              <p className="font-medium">{customer.lifecycle}</p>
            </div>
          </div>
          
          <div>
            <p className="text-gray-500 text-sm mb-1">Tags</p>
            <div className="flex flex-wrap gap-1">
              {customer.tags?.map(tag => (
                <span key={tag} className="px-2 py-1 bg-blue-100 text-blue-800 text-xs rounded">
                  {tag}
                </span>
              ))}
            </div>
          </div>
          
          <div className="text-xs text-gray-500">
            Renewal: {customer.customAttributes?.renewalDate}
          </div>
        </div>
      )}
    </div>
  );
};

const ExecutionMonitor = () => {
  const { realtimeData, connected, connectionError } = useWebSocket();
  const [executions, setExecutions] = useState(mockExecutions);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  // Use useCallback for async state updates
  const updateExecutions = useCallback((executionUpdate) => {
    setExecutions(prev => prev.map(exec => 
      exec.id === executionUpdate.executionId
        ? { ...exec, progress: executionUpdate.progress }
        : exec
    ));
  }, []);

  useEffect(() => {
    // Handle real-time execution updates with proper async handling
    if (realtimeData.execution_update) {
      setLoading(true);
      try {
        // Simulate async operation
        const timeoutId = setTimeout(() => {
          updateExecutions(realtimeData.execution_update);
          setLoading(false);
          setError(null);
        }, 100);

        return () => clearTimeout(timeoutId);
      } catch (err) {
        setError(err.message);
        setLoading(false);
      }
    }
  }, [realtimeData.execution_update, updateExecutions]);

  const getStatusIcon = (status) => {
    const icons = {
      'Running': '▶️',
      'Completed': '✅',
      'Paused': '⏸️',
      'Failed': '❌',
      'Scheduled': '⏰'
    };
    return icons[status] || '⚪';
  };

  const getStatusColor = (status) => {
    const colors = {
      'Running': 'bg-blue-100 text-blue-800',
      'Completed': 'bg-green-100 text-green-800',
      'Paused': 'bg-yellow-100 text-yellow-800',
      'Failed': 'bg-red-100 text-red-800',
      'Scheduled': 'bg-purple-100 text-purple-800'
    };
    return colors[status] || 'bg-gray-100 text-gray-800';
  };

  const runningCount = executions.filter(e => e.status === 'Running').length;

  if (error) {
    return (
      <div className="bg-white rounded-lg shadow-md p-6">
        <div className="text-center">
          <div className="text-red-600 mb-2">⚠️ Error loading executions</div>
          <p className="text-sm text-gray-600">{error}</p>
          <button 
            onClick={() => setError(null)}
            className="mt-2 px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
          >
            Retry
          </button>
        </div>
      </div>
    );
  }

  return (
    <div className="bg-white rounded-lg shadow-md p-6">
      <div className="flex justify-between items-center mb-6">
        <h3 className="text-lg font-semibold text-gray-800">Workflow Executions</h3>
        <div className="flex items-center gap-4">
          <div className="relative">
            <span className="text-xl">🔔</span>
            {runningCount > 0 && (
              <span className="absolute -top-2 -right-2 bg-blue-600 text-white text-xs rounded-full w-5 h-5 flex items-center justify-center">
                {runningCount}
              </span>
            )}
          </div>
          <div className="flex items-center gap-2 text-sm">
            <div className={`w-2 h-2 rounded-full ${connected ? 'bg-green-500' : 'bg-red-500'}`}></div>
            <span className={connected ? 'text-green-600' : 'text-red-600'}>
              {connected ? 'Live' : 'Offline'}
            </span>
          </div>
          {loading && <div className="text-blue-600">🔄</div>}
          <button className="text-blue-600 hover:underline text-sm">
            View All
          </button>
        </div>
      </div>

      {connectionError && (
        <div className="mb-4 p-3 bg-yellow-50 border border-yellow-200 rounded">
          <div className="text-yellow-800 text-sm">
            ⚠️ Connection issue: {connectionError}
          </div>
        </div>
      )}
      
      <div className="space-y-4">
        {executions.map((execution) => (
          <div key={execution.id} className="border rounded-lg p-4 hover:bg-gray-50 transition-colors">
            <div className="flex justify-between items-start mb-3">
              <div>
                <h4 className="font-medium text-gray-800">{execution.playbookName}</h4>
                <p className="text-sm text-gray-600">{execution.customer}</p>
              </div>
              <div className="flex items-center gap-2">
                <span>{getStatusIcon(execution.status)}</span>
                <span className={`px-2 py-1 rounded-full text-xs font-medium ${getStatusColor(execution.status)}`}>
                  {execution.status}
                </span>
              </div>
            </div>
            
            <div className="flex items-center gap-3 mb-3">
              <div className="flex-1 bg-gray-200 rounded-full h-2">
                <div 
                  className="bg-blue-500 h-2 rounded-full transition-all duration-300"
                  style={{ width: `${execution.progress}%` }}
                ></div>
              </div>
              <span className="text-sm font-medium">{execution.progress}%</span>
            </div>
            
            <div className="flex justify-between items-center text-sm text-gray-600">
              <div>
                <span>Current: {execution.currentNode}</span>
                {execution.errorMessage && (
                  <p className="text-red-600 mt-1">{execution.errorMessage}</p>
                )}
              </div>
              <div className="text-right">
                <p>Started: {formatDate(execution.startTime)}</p>
                <p>By: {execution.executedBy}</p>
              </div>
            </div>
            
            <div className="flex gap-2 mt-3">
              <button className="px-3 py-1 text-blue-600 border border-blue-600 rounded hover:bg-blue-50 text-sm transition-colors">
                View Details
              </button>
              {execution.status === 'Running' && (
                <button className="px-3 py-1 text-orange-600 border border-orange-600 rounded hover:bg-orange-50 text-sm transition-colors">
                  Pause
                </button>
              )}
              {execution.status === 'Paused' && (
                <button className="px-3 py-1 text-green-600 border border-green-600 rounded hover:bg-green-50 text-sm transition-colors">
                  Resume
                </button>
              )}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
};

const AdvancedDashboard = () => {
  const { currentOrg } = useOrganization();
  const { user } = useAuth();
  const { realtimeData, connected, connectionError } = useWebSocket();
  
  const [timeRange, setTimeRange] = useState('7d');
  const [customers, setCustomers] = useState(mockCustomers);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  // Use useCallback for async state updates
  const updateCustomerHealthScore = useCallback((customerId, newScore) => {
    setCustomers(prev => prev.map(customer => 
      customer.id === customerId
        ? { ...customer, healthScore: newScore }
        : customer
    ));
  }, []);

  // Real-time health score updates with proper async handling
  useEffect(() => {
    if (realtimeData.health_score_change) {
      setLoading(true);
      try {
        // Simulate async operation with proper cleanup
        const timeoutId = setTimeout(() => {
          updateCustomerHealthScore(
            realtimeData.health_score_change.customerId,
            realtimeData.health_score_change.newScore
          );
          setLoading(false);
          setError(null);
        }, 100);

        return () => clearTimeout(timeoutId);
      } catch (err) {
        setError(err.message);
        setLoading(false);
      }
    }
  }, [realtimeData.health_score_change, updateCustomerHealthScore]);

  const metrics = useMemo(() => {
    try {
      const totalCustomers = customers.length;
      const avgHealthScore = totalCustomers > 0 
        ? Math.round(customers.reduce((sum, c) => sum + (c.healthScore || 0), 0) / totalCustomers)
        : 0;
      const atRiskCustomers = customers.filter(c => (c.churnProbability || 0) > 0.7).length;
      const healthyCustomers = customers.filter(c => (c.healthScore || 0) >= 80).length;
      const totalMRR = customers.reduce((sum, c) => sum + (c.customAttributes?.mrr || 0), 0);
      const avgNPS = totalCustomers > 0
        ? Math.round(customers.reduce((sum, c) => sum + (c.npsScore || 0), 0) / totalCustomers * 10) / 10
        : 0;
      
      return {
        totalCustomers,
        avgHealthScore,
        atRiskCustomers,
        healthyCustomers,
        totalMRR,
        avgNPS,
        churnRate: totalCustomers > 0 ? Math.round((atRiskCustomers / totalCustomers) * 100) : 0,
        healthyRate: totalCustomers > 0 ? Math.round((healthyCustomers / totalCustomers) * 100) : 0
      };
    } catch (err) {
      console.error('Metrics calculation error:', err);
      return {
        totalCustomers: 0,
        avgHealthScore: 0,
        atRiskCustomers: 0,
        healthyCustomers: 0,
        totalMRR: 0,
        avgNPS: 0,
        churnRate: 0,
        healthyRate: 0
      };
    }
  }, [customers]);

  if (error) {
    return (
      <div className="space-y-6">
        <div className="bg-red-50 border border-red-200 rounded-lg p-6">
          <div className="text-center">
            <div className="text-red-600 mb-2">⚠️ Dashboard Error</div>
            <p className="text-sm text-gray-600">{error}</p>
            <button 
              onClick={() => setError(null)}
              className="mt-2 px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700"
            >
              Retry
            </button>
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <div>
          <h2 className="text-3xl font-bold text-gray-800">Customer Success Dashboard</h2>
          <div className="flex items-center gap-2">
            <p className="text-gray-600">Welcome back, {user?.name} - {currentOrg?.name}</p>
            {loading && <div className="text-blue-600">🔄</div>}
            {!connected && (
              <div className="flex items-center gap-1 text-red-600 text-sm">
                <div className="w-2 h-2 bg-red-500 rounded-full"></div>
                Offline
              </div>
            )}
          </div>
        </div>
        <div className="flex gap-2">
          {['24h', '7d', '30d', '90d'].map(range => (
            <button
              key={range}
              onClick={() => setTimeRange(range)}
              className={`px-3 py-1 rounded text-sm transition-colors ${
                timeRange === range 
                  ? 'bg-blue-600 text-white' 
                  : 'bg-gray-100 hover:bg-gray-200'
              }`}
            >
              {range}
            </button>
          ))}
        </div>
      </div>

      {connectionError && (
        <div className="bg-yellow-50 border border-yellow-200 rounded-lg p-4">
          <div className="text-yellow-800 text-sm">
            ⚠️ Real-time updates unavailable: {connectionError}
          </div>
        </div>
      )}
      
      {/* Enhanced KPI Cards */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
        <div className="bg-gradient-to-r from-blue-500 to-blue-600 text-white rounded-lg shadow-md p-6">
          <div className="flex items-center justify-between">
            <div>
              <div className="text-3xl font-bold">{metrics.avgHealthScore}</div>
              <div className="text-blue-100">Avg Health Score</div>
              <div className="text-sm text-blue-200 mt-1">
                {metrics.healthyRate}% customers healthy
              </div>
            </div>
            <span className="text-4xl opacity-80">💚</span>
          </div>
        </div>

        <div className="bg-gradient-to-r from-green-500 to-green-600 text-white rounded-lg shadow-md p-6">
          <div className="flex items-center justify-between">
            <div>
              <div className="text-3xl font-bold">${metrics.totalMRR.toLocaleString()}</div>
              <div className="text-green-100">Monthly Recurring Revenue</div>
              <div className="text-sm text-green-200 mt-1">
                +12% from last month
              </div>
            </div>
            <span className="text-4xl opacity-80">💰</span>
          </div>
        </div>

        <div className="bg-gradient-to-r from-purple-500 to-purple-600 text-white rounded-lg shadow-md p-6">
          <div className="flex items-center justify-between">
            <div>
              <div className="text-3xl font-bold">{mockExecutions.filter(e => e.status === 'Running').length}</div>
              <div className="text-purple-100">Active Workflows</div>
              <div className="text-sm text-purple-200 mt-1">
                {mockPlaybooks.filter(p => p.isActive).length} playbooks enabled
              </div>
            </div>
            <span className="text-4xl opacity-80">⚡</span>
          </div>
        </div>

        <div className="bg-gradient-to-r from-red-500 to-red-600 text-white rounded-lg shadow-md p-6">
          <div className="flex items-center justify-between">
            <div>
              <div className="text-3xl font-bold">{metrics.atRiskCustomers}</div>
              <div className="text-red-100">At Risk Customers</div>
              <div className="text-sm text-red-200 mt-1">
                {metrics.churnRate}% churn risk
              </div>
            </div>
            <span className="text-4xl opacity-80">⚠️</span>
          </div>
        </div>
      </div>

      {/* Customer Health Overview */}
      <div>
        <div className="flex justify-between items-center mb-4">
          <h3 className="text-xl font-semibold text-gray-800">Customer Health Overview</h3>
          <button className="text-blue-600 hover:underline text-sm">
            Manage Customers →
          </button>
        </div>
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
          {customers.map((customer) => (
            <AdvancedHealthScoreCard key={customer.id} customer={customer} />
          ))}
        </div>
      </div>

      {/* Execution Monitor */}
      <ExecutionMonitor />
    </div>
  );
};

// ========================================
// ENHANCED WORKFLOW DESIGNER
// ========================================

const EnhancedWorkflowDesigner = () => {
  const [nodes, setNodes] = useState([
    { id: '1', type: 'manual', x: 150, y: 100, config: { name: 'Customer Signup Trigger' } },
    { id: '2', type: 'email', x: 400, y: 100, config: { subject: 'Welcome!', template: 'welcome_template' } },
    { id: '3', type: 'wait', x: 650, y: 100, config: { duration: '1 day' } },
    { id: '4', type: 'condition', x: 900, y: 100, config: { condition: 'email_opened == true' } },
    { id: '5', type: 'sentiment', x: 1150, y: 50, config: { model: 'vader' } },
    { id: '6', type: 'salesforce', x: 1150, y: 150, config: { operation: 'update_contact' } }
  ]);
  
  const [selectedNode, setSelectedNode] = useState(null);
  const [editDialogOpen, setEditDialogOpen] = useState(false);
  const [editingNode, setEditingNode] = useState(null);
  const [selectedCategory, setSelectedCategory] = useState('all');
  const [dragState, setDragState] = useState({ isDragging: false, nodeId: null });

  const categories = ['all', 'triggers', 'communication', 'logic', 'data', 'ai', 'integrations'];

  const filteredNodeTypes = selectedCategory === 'all' 
    ? NodeTypes 
    : Object.fromEntries(
        Object.entries(NodeTypes).filter(([_, config]) => config.category === selectedCategory)
      );

  const handleEditNode = useCallback((node) => {
    setEditingNode(node);
    setEditDialogOpen(true);
  }, []);

  const handleDeleteNode = useCallback((nodeId) => {
    setNodes(prev => prev.filter(n => n.id !== nodeId));
    if (selectedNode?.id === nodeId) {
      setSelectedNode(null);
    }
  }, [selectedNode]);

  const addNode = useCallback((type) => {
    const newNode = {
      id: generateId(),
      type,
      x: Math.random() * 500 + 200,
      y: Math.random() * 300 + 200,
      config: {}
    };
    setNodes(prev => [...prev, newNode]);
  }, []);

  const updateNodePosition = useCallback((nodeId, x, y) => {
    setNodes(prev => prev.map(node => 
      node.id === nodeId ? { ...node, x, y } : node
    ));
  }, []);

  const saveNodeConfig = useCallback((nodeId, config) => {
    setNodes(prev => prev.map(node => 
      node.id === nodeId ? { ...node, config } : node
    ));
  }, []);

  const connections = useMemo(() => {
    // Create connections between adjacent nodes
    return nodes.slice(0, -1).map((node, index) => ({
      from: node.id,
      to: nodes[index + 1].id,
      fromX: node.x + 90,
      fromY: node.y,
      toX: nodes[index + 1].x - 90,
      toY: nodes[index + 1].y
    }));
  }, [nodes]);

  // Enhanced drag handling with better state management
  const handleMouseDown = useCallback((e, node) => {
    e.preventDefault();
    setDragState({ isDragging: true, nodeId: node.id });
    setSelectedNode(node);
    
    const startX = e.clientX - node.x;
    const startY = e.clientY - node.y;
    
    const handleMouseMove = (e) => {
      if (dragState.isDragging && dragState.nodeId === node.id) {
        updateNodePosition(node.id, e.clientX - startX, e.clientY - startY);
      }
    };
    
    const handleMouseUp = () => {
      setDragState({ isDragging: false, nodeId: null });
      document.removeEventListener('mousemove', handleMouseMove);
      document.removeEventListener('mouseup', handleMouseUp);
    };
    
    document.addEventListener('mousemove', handleMouseMove);
    document.addEventListener('mouseup', handleMouseUp);
  }, [dragState, updateNodePosition]);

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h2 className="text-2xl font-bold text-gray-800">Visual Workflow Designer</h2>
        <div className="flex gap-2">
          <button className="bg-green-600 text-white px-4 py-2 rounded-lg hover:bg-green-700 transition-colors">
            Save Workflow
          </button>
          <button className="bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700 transition-colors">
            Test Run
          </button>
        </div>
      </div>
      
      {/* Node Category Filter */}
      <div className="flex gap-2 mb-4">
        {categories.map(category => (
          <button
            key={category}
            onClick={() => setSelectedCategory(category)}
            className={`px-3 py-1 rounded text-sm capitalize transition-colors ${
              selectedCategory === category 
                ? 'bg-blue-600 text-white' 
                : 'bg-gray-100 hover:bg-gray-200'
            }`}
          >
            {category}
          </button>
        ))}
      </div>

      {/* Node Palette */}
      <div className="bg-white rounded-lg shadow-md p-4">
        <h3 className="font-semibold text-gray-700 mb-3">Node Palette</h3>
        <div className="grid grid-cols-2 md:grid-cols-4 lg:grid-cols-6 gap-3">
          {Object.entries(filteredNodeTypes).map(([type, config]) => (
            <button
              key={type}
              onClick={() => addNode(type)}
              className={`p-3 rounded-lg border-2 text-white font-medium hover:opacity-80 transition-opacity text-sm ${config.color}`}
              title={config.description}
            >
              <div className="text-center">
                <div className="text-xl mb-1">{config.icon}</div>
                <div className="text-xs">{config.label}</div>
              </div>
            </button>
          ))}
        </div>
      </div>

      {/* Workflow Canvas */}
      <div className="bg-gray-50 rounded-lg border shadow-md">
        <div className="p-4 border-b bg-white rounded-t-lg">
          <h3 className="text-lg font-semibold text-gray-700">Workflow Canvas</h3>
          <p className="text-sm text-gray-600">Drag nodes to reposition them</p>
        </div>
        
        <div className="p-6 relative h-[600px] overflow-auto select-none">
          {/* Connection lines */}
          <svg className="absolute top-0 left-0 w-full h-full pointer-events-none z-10">
            {connections.map((connection, index) => (
              <g key={`connection-${index}`}>
                <line
                  x1={connection.fromX}
                  y1={connection.fromY}
                  x2={connection.toX}
                  y2={connection.toY}
                  stroke="#1976d2"
                  strokeWidth="2"
                  markerEnd="url(#arrowhead)"
                />
              </g>
            ))}
            <defs>
              <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
                <polygon points="0 0, 10 3.5, 0 7" fill="#1976d2" />
              </marker>
            </defs>
          </svg>

          {/* Workflow Nodes */}
          {nodes.map((node) => (
            <div
              key={node.id}
              className="absolute z-20"
              style={{ 
                left: node.x - 90, 
                top: node.y - 50,
                transform: 'translate(0, 0)'
              }}
            >
              <div
                className={`bg-white rounded-lg shadow-md p-4 cursor-move border-2 transition-all hover:shadow-lg min-w-[180px] ${
                  selectedNode?.id === node.id ? 'border-blue-500 ring-2 ring-blue-200' : 'border-gray-200'
                } ${dragState.isDragging && dragState.nodeId === node.id ? 'opacity-75' : ''}`}
                onClick={() => setSelectedNode(node)}
                onMouseDown={(e) => handleMouseDown(e, node)}
              >
                <div className="flex items-center gap-2 mb-2">
                  <span className="text-xl">{NodeTypes[node.type]?.icon}</span>
                  <span className="font-semibold text-gray-800">{NodeTypes[node.type]?.label}</span>
                </div>
                <p className="text-xs text-gray-600 mb-3">{NodeTypes[node.type]?.description}</p>
                
                {/* Node configuration preview */}
                {node.config && Object.keys(node.config).length > 0 && (
                  <div className="text-xs text-gray-500 mb-2 p-2 bg-gray-50 rounded">
                    {Object.entries(node.config).slice(0, 2).map(([key, value]) => (
                      <div key={key}>{key}: {String(value).substring(0, 20)}...</div>
                    ))}
                  </div>
                )}
                
                <div className="flex gap-2">
                  <button 
                    onClick={(e) => { e.stopPropagation(); handleEditNode(node); }}
                    className="p-1 text-blue-600 hover:bg-blue-50 rounded transition-colors"
                    title="Edit node"
                  >
                    ✏️
                  </button>
                  <button 
                    onClick={(e) => { e.stopPropagation(); handleDeleteNode(node.id); }}
                    className="p-1 text-red-600 hover:bg-red-50 rounded transition-colors"
                    title="Delete node"
                  >
                    🗑️
                  </button>
                </div>
              </div>
            </div>
          ))}
        </div>
      </div>

      {/* Node Properties Panel */}
      {selectedNode && (
        <div className="bg-white rounded-lg shadow-md p-6">
          <h3 className="text-lg font-semibold text-gray-800 mb-4">
            Node Properties: {NodeTypes[selectedNode.type]?.label}
          </h3>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">Node ID</label>
              <input 
                type="text" 
                value={selectedNode.id} 
                disabled 
                className="w-full p-2 border rounded bg-gray-50"
              />
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">Type</label>
              <input 
                type="text" 
                value={NodeTypes[selectedNode.type]?.label} 
                disabled 
                className="w-full p-2 border rounded bg-gray-50"
              />
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">Position</label>
              <input 
                type="text" 
                value={`${Math.round(selectedNode.x)}, ${Math.round(selectedNode.y)}`} 
                disabled 
                className="w-full p-2 border rounded bg-gray-50"
              />
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">Category</label>
              <input 
                type="text" 
                value={NodeTypes[selectedNode.type]?.category} 
                disabled 
                className="w-full p-2 border rounded bg-gray-50"
              />
            </div>
          </div>
        </div>
      )}

      {/* Enhanced Edit Node Modal */}
      {editDialogOpen && editingNode && (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
          <div className="bg-white rounded-lg p-6 w-full max-w-2xl max-h-[80vh] overflow-y-auto">
            <h3 className="text-lg font-semibold mb-4">
              Configure {NodeTypes[editingNode.type]?.label} Node
            </h3>
            
            <div className="space-y-4">
              {/* Email Node Configuration */}
              {editingNode.type === 'email' && (
                <>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Email Subject</label>
                    <input
                      type="text"
                      placeholder="Enter email subject"
                      className="w-full p-3 border rounded-lg"
                      defaultValue={editingNode.config?.subject || "Welcome to our platform!"}
                    />
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Template</label>
                    <select className="w-full p-3 border rounded-lg">
                      <option value="welcome_template">Welcome Template</option>
                      <option value="onboarding_template">Onboarding Template</option>
                      <option value="renewal_template">Renewal Template</option>
                      <option value="custom_template">Custom Template</option>
                    </select>
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Personalization</label>
                    <div className="grid grid-cols-2 gap-2">
                      <label className="flex items-center">
                        <input type="checkbox" className="mr-2" defaultChecked />
                        Include customer name
                      </label>
                      <label className="flex items-center">
                        <input type="checkbox" className="mr-2" />
                        Include company name
                      </label>
                    </div>
                  </div>
                </>
              )}

              {/* Condition Node Configuration */}
              {editingNode.type === 'condition' && (
                <>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Condition Expression</label>
                    <textarea
                      placeholder="Enter condition logic"
                      className="w-full p-3 border rounded-lg"
                      rows={3}
                      defaultValue={editingNode.config?.condition || "customer.health_score > 80"}
                    />
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">True Branch Action</label>
                    <select className="w-full p-3 border rounded-lg">
                      <option value="continue">Continue to next node</option>
                      <option value="branch_true">Go to true branch</option>
                      <option value="end">End workflow</option>
                    </select>
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">False Branch Action</label>
                    <select className="w-full p-3 border rounded-lg">
                      <option value="continue">Continue to next node</option>
                      <option value="branch_false">Go to false branch</option>
                      <option value="end">End workflow</option>
                    </select>
                  </div>
                </>
              )}

              {/* AI/ML Node Configuration */}
              {editingNode.type === 'sentiment' && (
                <>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Input Text Source</label>
                    <select className="w-full p-3 border rounded-lg">
                      <option value="customer_email">Customer Email Content</option>
                      <option value="support_ticket">Support Ticket Text</option>
                      <option value="survey_response">Survey Response</option>
                      <option value="custom_field">Custom Field</option>
                    </select>
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Sentiment Model</label>
                    <select className="w-full p-3 border rounded-lg">
                      <option value="vader">VADER Sentiment</option>
                      <option value="textblob">TextBlob</option>
                      <option value="openai">OpenAI GPT</option>
                      <option value="huggingface">HuggingFace Transformers</option>
                    </select>
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Confidence Threshold</label>
                    <input
                      type="range"
                      min="0"
                      max="1"
                      step="0.1"
                      defaultValue="0.7"
                      className="w-full"
                    />
                    <div className="text-sm text-gray-600">Current: 0.7</div>
                  </div>
                </>
              )}

              {/* Integration Node Configuration */}
              {editingNode.type === 'salesforce' && (
                <>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Operation</label>
                    <select className="w-full p-3 border rounded-lg">
                      <option value="create_contact">Create Contact</option>
                      <option value="update_contact">Update Contact</option>
                      <option value="create_opportunity">Create Opportunity</option>
                      <option value="update_account">Update Account</option>
                    </select>
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Field Mapping</label>
                    <div className="space-y-2">
                      <div className="grid grid-cols-2 gap-2">
                        <input type="text" placeholder="Platform Field" className="p-2 border rounded" />
                        <input type="text" placeholder="Salesforce Field" className="p-2 border rounded" />
                      </div>
                      <button className="text-blue-600 text-sm hover:underline">+ Add Field Mapping</button>
                    </div>
                  </div>
                </>
              )}

              {/* Wait Node Configuration */}
              {editingNode.type === 'wait' && (
                <>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Wait Duration</label>
                    <div className="grid grid-cols-2 gap-2">
                      <input
                        type="number"
                        placeholder="Duration"
                        className="p-3 border rounded-lg"
                        defaultValue="1"
                      />
                      <select className="p-3 border rounded-lg">
                        <option value="minutes">Minutes</option>
                        <option value="hours">Hours</option>
                        <option value="days">Days</option>
                        <option value="weeks">Weeks</option>
                      </select>
                    </div>
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">Wait Type</label>
                    <select className="w-full p-3 border rounded-lg">
                      <option value="fixed">Fixed Duration</option>
                      <option value="business_hours">Business Hours Only</option>
                      <option value="until_condition">Until Condition Met</option>
                    </select>
                  </div>
                </>
              )}
            </div>
            
            <div className="flex gap-2 mt-6">
              <button
                onClick={() => setEditDialogOpen(false)}
                className="px-4 py-2 border rounded-lg hover:bg-gray-50 transition-colors"
              >
                Cancel
              </button>
              <button
                onClick={() => {
                  // Save node configuration logic here
                  setEditDialogOpen(false);
                }}
                className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors"
              >
                Save Configuration
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

// ========================================
// ENHANCED PLAYBOOK LIBRARY
// ========================================

const EnhancedPlaybookLibrary = () => {
  const [playbooks, setPlaybooks] = useState(mockPlaybooks);
  const [createDialogOpen, setCreateDialogOpen] = useState(false);
  const [filterCategory, setFilterCategory] = useState('all');
  const [sortBy, setSortBy] = useState('name');
  const [searchTerm, setSearchTerm] = useState('');

  const filteredPlaybooks = useMemo(() => {
    let filtered = playbooks;
    
    if (filterCategory !== 'all') {
      filtered = filtered.filter(p => p.category.toLowerCase() === filterCategory);
    }
    
    if (searchTerm) {
      filtered = filtered.filter(p => 
        p.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
        p.description.toLowerCase().includes(searchTerm.toLowerCase())
      );
    }
    
    return filtered.sort((a, b) => {
      switch (sortBy) {
        case 'name': return a.name.localeCompare(b.name);
        case 'category': return a.category.localeCompare(b.category);
        case 'success_rate': return (b.successRate || 0) - (a.successRate || 0);
        case 'executions': return (b.totalExecutions || 0) - (a.totalExecutions || 0);
        case 'last_run': return new Date(b.lastRun || 0) - new Date(a.lastRun || 0);
        default: return 0;
      }
    });
  }, [playbooks, filterCategory, sortBy, searchTerm]);

  const categories = ['all', 'onboarding', 'monitoring', 'retention', 'growth'];

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h2 className="text-3xl font-bold text-gray-800">Playbook Library</h2>
        <button 
          onClick={() => setCreateDialogOpen(true)}
          className="bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700 flex items-center gap-2"
        >
          <span>➕</span>
          Create Playbook
        </button>
      </div>

      {/* Filters and Search */}
      <div className="bg-white rounded-lg shadow-md p-6">
        <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">Search</label>
            <input
              type="text"
              placeholder="Search playbooks..."
              value={searchTerm}
              onChange={(e) => setSearchTerm(e.target.value)}
              className="w-full p-2 border rounded-lg"
            />
          </div>
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">Category</label>
            <select
              value={filterCategory}
              onChange={(e) => setFilterCategory(e.target.value)}
              className="w-full p-2 border rounded-lg"
            >
              {categories.map(cat => (
                <option key={cat} value={cat} className="capitalize">
                  {cat === 'all' ? 'All Categories' : cat}
                </option>
              ))}
            </select>
          </div>
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">Sort By</label>
            <select
              value={sortBy}
              onChange={(e) => setSortBy(e.target.value)}
              className="w-full p-2 border rounded-lg"
            >
              <option value="name">Name</option>
              <option value="category">Category</option>
              <option value="success_rate">Success Rate</option>
              <option value="executions">Total Executions</option>
              <option value="last_run">Last Run</option>
            </select>
          </div>
          <div className="flex items-end">
            <button 
              onClick={() => {
                setSearchTerm('');
                setFilterCategory('all');
                setSortBy('name');
              }}
              className="w-full p-2 bg-gray-100 hover:bg-gray-200 rounded-lg"
            >
              Reset Filters
            </button>
          </div>
        </div>
      </div>

      {/* Playbook Statistics */}
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
        <div className="bg-blue-50 p-4 rounded-lg">
          <div className="text-2xl font-bold text-blue-600">{playbooks.length}</div>
          <div className="text-sm text-blue-600">Total Playbooks</div>
        </div>
        <div className="bg-green-50 p-4 rounded-lg">
          <div className="text-2xl font-bold text-green-600">{playbooks.filter(p => p.isActive).length}</div>
          <div className="text-sm text-green-600">Active Playbooks</div>
        </div>
        <div className="bg-purple-50 p-4 rounded-lg">
          <div className="text-2xl font-bold text-purple-600">
            {Math.round(playbooks.reduce((sum, p) => sum + (p.successRate || 0), 0) / playbooks.length)}%
          </div>
          <div className="text-sm text-purple-600">Avg Success Rate</div>
        </div>
        <div className="bg-orange-50 p-4 rounded-lg">
          <div className="text-2xl font-bold text-orange-600">
            {playbooks.reduce((sum, p) => sum + (p.totalExecutions || 0), 0).toLocaleString()}
          </div>
          <div className="text-sm text-orange-600">Total Executions</div>
        </div>
      </div>

      {/* Playbook Grid */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {filteredPlaybooks.map((playbook) => (
          <div key={playbook.id} className="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow">
            <div className="flex justify-between items-start mb-4">
              <div>
                <h3 className="text-lg font-semibold text-gray-800 mb-1">{playbook.name}</h3>
                <span className="text-sm text-gray-600">{playbook.category}</span>
              </div>
              <div className="flex items-center gap-2">
                <span className={`px-2 py-1 rounded-full text-xs font-medium ${
                  playbook.isActive ? 'bg-green-100 text-green-800' : 'bg-gray-100 text-gray-800'
                }`}>
                  {playbook.isActive ? 'Active' : 'Inactive'}
                </span>
                <button className="text-gray-400 hover:text-gray-600">
                  ⋮
                </button>
              </div>
            </div>

            <p className="text-sm text-gray-600 mb-4 line-clamp-2">
              {playbook.description}
            </p>

            <div className="grid grid-cols-2 gap-4 mb-4 text-sm">
              <div>
                <span className="text-gray-500">Nodes:</span>
                <span className="font-medium ml-1">{playbook.nodes}</span>
              </div>
              <div>
                <span className="text-gray-500">Triggers:</span>
                <span className="font-medium ml-1">{playbook.triggers}</span>
              </div>
              <div>
                <span className="text-gray-500">Success Rate:</span>
                <span className={`font-medium ml-1 ${
                  playbook.successRate >= 90 ? 'text-green-600' : 
                  playbook.successRate >= 80 ? 'text-yellow-600' : 'text-red-600'
                }`}>
                  {playbook.successRate}%
                </span>
              </div>
              <div>
                <span className="text-gray-500">Executions:</span>
                <span className="font-medium ml-1">{playbook.totalExecutions?.toLocaleString()}</span>
              </div>
            </div>

            <div className="text-xs text-gray-500 mb-4">
              <div>Created by: {playbook.createdBy}</div>
              <div>Version: {playbook.version}</div>
              <div>Last run: {playbook.lastRun}</div>
              <div>Avg duration: {playbook.avgExecutionTime}</div>
            </div>

            <div className="flex gap-2">
              <button className="flex-1 px-3 py-1 border border-gray-300 rounded text-sm hover:bg-gray-50">
                Edit
              </button>
              <button className="flex-1 px-3 py-1 bg-blue-600 text-white rounded text-sm hover:bg-blue-700 flex items-center justify-center gap-1">
                <span>▶️</span>
                Run
              </button>
              <button className="px-3 py-1 border border-gray-300 rounded text-sm hover:bg-gray-50">
                📊
              </button>
            </div>
          </div>
        ))}
      </div>

      {/* Create Playbook Modal */}
      {createDialogOpen && (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
          <div className="bg-white rounded-lg p-6 w-full max-w-2xl">
            <h3 className="text-lg font-semibold mb-4">Create New Playbook</h3>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div className="space-y-4">
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">Playbook Name</label>
                  <input
                    type="text"
                    placeholder="Enter playbook name"
                    className="w-full p-3 border rounded-lg"
                  />
                </div>
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">Category</label>
                  <select className="w-full p-3 border rounded-lg">
                    <option value="">Select Category</option>
                    <option value="Onboarding">Onboarding</option>
                    <option value="Retention">Retention</option>
                    <option value="Growth">Growth</option>
                    <option value="Monitoring">Monitoring</option>
                  </select>
                </div>
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">Trigger Type</label>
                  <select className="w-full p-3 border rounded-lg">
                    <option value="manual">Manual</option>
                    <option value="webhook">Webhook</option>
                    <option value="scheduled">Scheduled</option>
                    <option value="event">Event-based</option>
                  </select>
                </div>
              </div>
              <div className="space-y-4">
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">Description</label>
                  <textarea
                    placeholder="Describe the playbook purpose and workflow"
                    className="w-full p-3 border rounded-lg"
                    rows={4}
                  />
                </div>
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">Tags</label>
                  <input
                    type="text"
                    placeholder="Enter tags separated by commas"
                    className="w-full p-3 border rounded-lg"
                  />
                </div>
                <div>
                  <label className="flex items-center">
                    <input type="checkbox" className="mr-2" defaultChecked />
                    <span className="text-sm">Activate immediately</span>
                  </label>
                </div>
              </div>
            </div>
            <div className="flex gap-2 mt-6">
              <button
                onClick={() => setCreateDialogOpen(false)}
                className="px-4 py-2 border rounded-lg hover:bg-gray-50"
              >
                Cancel
              </button>
              <button
                onClick={() => setCreateDialogOpen(false)}
                className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
              >
                Create Playbook
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

// ========================================
// MAIN APPLICATION COMPONENT
// ========================================

const CustomerSuccessPlatform = () => {
  const [activeTab, setActiveTab] = useState(0);

  const menuItems = [
    { label: 'Dashboard', icon: '📊', component: <AdvancedDashboard /> },
    { label: 'Playbooks', icon: '▶️', component: <EnhancedPlaybookLibrary /> },
    { label: 'Workflow Designer', icon: '✏️', component: <EnhancedWorkflowDesigner /> },
    { label: 'Customers', icon: '👥', component: <div className="p-8 text-center text-gray-600">Advanced Customer Management (In Development)</div> },
    { label: 'Analytics', icon: '📈', component: <div className="p-8 text-center text-gray-600">Advanced Analytics & Reporting (In Development)</div> },
    { label: 'Integrations', icon: '🔗', component: <div className="p-8 text-center text-gray-600">External Integrations Management (In Development)</div> },
    { label: 'Settings', icon: '⚙️', component: <div className="p-8 text-center text-gray-600">Organization Settings (In Development)</div> }
  ];

  return (
    <AuthProvider>
      <OrganizationProvider>
        <WebSocketProvider>
          <CustomerSuccessPlatformInner activeTab={activeTab} setActiveTab={setActiveTab} menuItems={menuItems} />
        </WebSocketProvider>
      </OrganizationProvider>
    </AuthProvider>
  );
};

const CustomerSuccessPlatformInner = ({ activeTab, setActiveTab, menuItems }) => {
  const { user, isAuthenticated } = useAuth();
  const { currentOrg } = useOrganization();
  const { notifications, connected } = useWebSocket();

  const unreadNotifications = notifications.filter(n => !n.read).length;

  if (!isAuthenticated) {
    return (
      <div className="min-h-screen bg-gray-100 flex items-center justify-center">
        <div className="bg-white p-8 rounded-lg shadow-md">
          <h1 className="text-2xl font-bold mb-4">Customer Success Platform</h1>
          <p className="text-gray-600">Please log in to continue</p>
        </div>
      </div>
    );
  }

  return (
    <div className="flex h-screen bg-gray-100">
      {/* Sidebar */}
      <div className="w-64 bg-white shadow-lg">
        <div className="p-6 border-b">
          <div className="flex items-center gap-2 mb-2">
            <span className="text-2xl">{currentOrg?.settings?.customBranding?.logo}</span>
            <h1 className="text-lg font-bold text-gray-800">
              {currentOrg?.name}
            </h1>
          </div>
          <div className="text-sm text-gray-600">
            {currentOrg?.subscriptionTier} Plan
          </div>
        </div>
        
        <nav className="mt-6">
          {menuItems.map((item, index) => (
            <button
              key={item.label}
              onClick={() => setActiveTab(index)}
              className={`w-full flex items-center gap-3 px-6 py-3 text-left hover:bg-blue-50 transition-colors ${
                activeTab === index ? 'bg-blue-50 border-r-2 border-blue-600 text-blue-600' : 'text-gray-700'
              }`}
            >
              <span className="text-lg">{item.icon}</span>
              <span className="font-medium">{item.label}</span>
            </button>
          ))}
        </nav>

        {/* Connection Status */}
        <div className="absolute bottom-4 left-4 right-4">
          <div className={`flex items-center gap-2 text-xs ${
            connected ? 'text-green-600' : 'text-red-600'
          }`}>
            <div className={`w-2 h-2 rounded-full ${
              connected ? 'bg-green-500' : 'bg-red-500'
            }`}></div>
            {connected ? 'Connected' : 'Disconnected'}
          </div>
        </div>
      </div>

      {/* Main Content */}
      <div className="flex-1 flex flex-col">
        {/* Header */}
        <header className="bg-white shadow-sm border-b px-6 py-4">
          <div className="flex justify-between items-center">
            <div>
              <h2 className="text-xl font-semibold text-gray-800">{menuItems[activeTab]?.label}</h2>
              <p className="text-sm text-gray-600">
                Welcome back, {user?.name} {user?.avatar}
              </p>
            </div>
            <div className="flex items-center gap-4">
              <div className="relative">
                <button className="relative p-2 text-gray-600 hover:text-gray-800">
                  <span className="text-xl">🔔</span>
                  {unreadNotifications > 0 && (
                    <span className="absolute -top-1 -right-1 bg-red-500 text-white text-xs rounded-full w-5 h-5 flex items-center justify-center">
                      {unreadNotifications}
                    </span>
                  )}
                </button>
              </div>
              <div className="flex items-center gap-2">
                <span className="text-sm font-medium">{user?.role}</span>
                <div className="w-8 h-8 bg-blue-100 rounded-full flex items-center justify-center">
                  {user?.avatar}
                </div>
              </div>
            </div>
          </div>
        </header>

        {/* Content Area */}
        <main className="flex-1 p-6 overflow-auto">
          {menuItems[activeTab]?.component}
        </main>
      </div>

      {/* Real-time Notifications */}
      {notifications.slice(-3).map((notification, index) => (
        <div
          key={notification.id}
          className="fixed bottom-4 right-4 bg-blue-600 text-white p-4 rounded-lg shadow-lg max-w-sm z-50"
          style={{ 
            bottom: `${16 + index * 80}px`,
            animationDuration: '0.3s' 
          }}
        >
          <div className="flex items-start gap-2">
            <span>🔔</span>
            <div className="flex-1">
              <p className="text-sm">{notification.message}</p>
              <p className="text-xs text-blue-200 mt-1">
                {formatDate(notification.timestamp)}
              </p>
            </div>
            <button className="text-white hover:text-gray-200 ml-2">
              ×
            </button>
          </div>
        </div>
      ))}
    </div>
  );
};

export default CustomerSuccessPlatform;
