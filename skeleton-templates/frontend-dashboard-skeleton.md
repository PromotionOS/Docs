# Frontend Dashboard — Skeleton Build Template

> Language: JavaScript (React 18)
> Build tool: Vite
> Styling: Tailwind CSS
> HTTP client: Axios
> Deploy: Vercel
> Status: Skeleton only — page scaffolds and API clients defined, no logic implemented

---

## Repo Structure

```
frontend-dashboard/
├── src/
│   ├── pages/
│   │   ├── MXDashboard.jsx
│   │   ├── CXView.jsx
│   │   └── AnalyticsDashboard.jsx
│   ├── components/
│   │   ├── CampaignList.jsx
│   │   ├── CampaignForm.jsx
│   │   ├── CampaignDetail.jsx
│   │   ├── OfferList.jsx
│   │   ├── BurnChart.jsx
│   │   ├── LiftChart.jsx
│   │   ├── AlertBanner.jsx
│   │   └── StatusBadge.jsx
│   ├── api/
│   │   ├── campaignApi.js
│   │   ├── eligibilityApi.js
│   │   ├── analyticsApi.js
│   │   └── redemptionApi.js
│   ├── constants/
│   │   └── testData.js
│   ├── App.jsx
│   └── main.jsx
├── public/
├── .env
├── .env.example
├── index.html
├── vite.config.js
├── tailwind.config.js
├── package.json
└── README.md
```

---

## package.json

```json
{
  "name": "frontend-dashboard",
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "axios": "^1.6.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "recharts": "^2.10.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.2.0",
    "autoprefixer": "^10.4.16",
    "postcss": "^8.4.32",
    "tailwindcss": "^3.3.6",
    "vite": "^5.0.0"
  }
}
```

---

## .env.example

```
VITE_CAMPAIGN_API_URL=https://campaign-service.railway.app
VITE_ELIGIBILITY_API_URL=https://eligibility-service.railway.app
VITE_ANALYTICS_API_URL=https://analytics-service.railway.app
VITE_REDEMPTION_API_URL=https://redemption-service.railway.app
VITE_TENANT_ID=tenant-kroger-001
```

---

## API Clients (Stubs)

### api/campaignApi.js

```javascript
import axios from 'axios'

const BASE = import.meta.env.VITE_CAMPAIGN_API_URL
const TENANT = import.meta.env.VITE_TENANT_ID

export const getCampaigns = async (status = null) => {
  // TODO Team 6 Sprint 1 — wire to live Campaign Service
  // GET /campaigns?tenantId=TENANT&status=status
  // For Sprint 1: return mock data from constants/testData.js
  return null
}

export const createCampaign = async (campaign) => {
  // TODO Team 6 Sprint 2 — POST /campaigns
  return null
}

export const addFunding = async (campaignId, funding) => {
  // TODO Team 6 Sprint 2 — POST /campaigns/:id/funding
  return null
}

export const publishCampaign = async (campaignId) => {
  // TODO Team 6 Sprint 2 — PUT /campaigns/:id/publish
  return null
}

export const getCampaignSummary = async (campaignId) => {
  // TODO Team 6 Sprint 3 — GET /campaigns/:id/summary
  return null
}

export const updateBudget = async (campaignId, totalAmount) => {
  // TODO Team 6 Sprint 3 — PUT /campaigns/:id/budget
  return null
}
```

### api/eligibilityApi.js

```javascript
import axios from 'axios'

const BASE = import.meta.env.VITE_ELIGIBILITY_API_URL
const TENANT = import.meta.env.VITE_TENANT_ID

export const checkEligibility = async (campaignId, customerId, cartTotal, cartUPCs = []) => {
  // TODO Team 6 Sprint 1 — POST /eligibility/check
  // Returns { eligible, campaignId, offerType, discountApplied, reason }
  return null
}

export const getOffers = async (customerId, cartTotal = 0) => {
  // TODO Team 6 Sprint 2 — GET /offers?customerId=X&tenantId=Y&cartTotal=Z
  // Returns { eligibleOffers: [], ineligibleOffers: [] }
  return null
}
```

### api/analyticsApi.js

```javascript
import axios from 'axios'

const BASE = import.meta.env.VITE_ANALYTICS_API_URL
const TENANT = import.meta.env.VITE_TENANT_ID

export const getCampaignReport = async (campaignId) => {
  // TODO Team 6 Sprint 2 — GET /analytics/campaigns/:id/report
  return null
}

export const getCampaignBurn = async (campaignId) => {
  // TODO Team 6 Sprint 2 — GET /analytics/campaigns/:id/burn
  // Used for real-time polling
  return null
}
```

### api/redemptionApi.js

```javascript
import axios from 'axios'

const BASE = import.meta.env.VITE_REDEMPTION_API_URL
const TENANT = import.meta.env.VITE_TENANT_ID

export const redeem = async (campaignId, customerId, discountApplied, cartTotal,
                              storeId, division) => {
  // TODO Team 6 Sprint 2
  // POST /redeem with idempotencyKey = `browser-${Date.now()}-${Math.random()}`
  return null
}
```

---

## constants/testData.js (Mock Data for Sprint 1)

```javascript
// Used by Team 6 Sprint 1 while Campaign Service is being built
export const MOCK_CAMPAIGNS = [
  {
    id: 'camp-001',
    name: 'Weekend Mega Sale',
    status: 'PAUSED',
    offerType: 'PCT_OFF',
    budgetBurnPercent: 95.2,
    dateRange: { startDate: '2026-06-01', endDate: '2026-06-30' }
  },
  {
    id: 'camp-002',
    name: 'Buy 2 Get 1 Free — Cereal',
    status: 'ACTIVE',
    offerType: 'BOGO',
    budgetBurnPercent: 40.0,
    dateRange: { startDate: '2026-06-01', endDate: '2026-06-15' }
  },
  {
    id: 'camp-003',
    name: 'Spend $50 Get $10 Off',
    status: 'ACTIVE',
    offerType: 'THRESHOLD',
    budgetBurnPercent: 23.0,
    dateRange: { startDate: '2026-06-01', endDate: '2026-06-30' }
  },
  {
    id: 'camp-004',
    name: 'Platinum Member Exclusive — 30% Off Wine',
    status: 'ACTIVE',
    offerType: 'PCT_OFF',
    budgetBurnPercent: 25.0,
    dateRange: { startDate: '2026-06-01', endDate: '2026-06-30' }
  },
  {
    id: 'camp-006',
    name: 'Pharmacy — Vitamin Bundle',
    status: 'PAUSED',
    offerType: 'PCT_OFF',
    budgetBurnPercent: 100.0,
    dateRange: { startDate: '2026-06-01', endDate: '2026-06-30' }
  },
  {
    id: 'camp-007',
    name: 'Back to School Stationery',
    status: 'SCHEDULED',
    offerType: 'PCT_OFF',
    budgetBurnPercent: 0.0,
    dateRange: { startDate: '2026-08-01', endDate: '2026-08-31' }
  }
]

export const MOCK_CUSTOMERS = [
  { id: 'cust-001', name: 'Alice Johnson', loyaltyTier: 'PLATINUM', division: 'division-midwest' },
  { id: 'cust-002', name: 'Bob Martinez', loyaltyTier: 'GOLD', division: 'division-southeast' },
  { id: 'cust-003', name: 'Carol White', loyaltyTier: 'SILVER', division: 'division-midwest' },
  { id: 'cust-004', name: 'David Kim', loyaltyTier: 'BASIC', division: 'division-southeast' },
  { id: 'cust-006', name: 'Frank Nguyen (GOLD boundary)', loyaltyTier: 'GOLD', division: 'division-west' },
  { id: 'cust-007', name: 'Grace Lee (PLATINUM boundary)', loyaltyTier: 'PLATINUM', division: 'division-west' },
  { id: 'cust-010', name: 'Jack Wilson (geo miss)', loyaltyTier: 'BASIC', division: 'division-southwest' },
  { id: 'cust-011', name: 'Karen Adams (multi-segment PLATINUM)', loyaltyTier: 'PLATINUM', division: 'division-midwest' }
]
```

---

## Page Scaffolds

### pages/MXDashboard.jsx

```jsx
import React, { useState, useEffect } from 'react'
import CampaignList from '../components/CampaignList'
import CampaignForm from '../components/CampaignForm'
import AlertBanner from '../components/AlertBanner'

export default function MXDashboard() {
  const [campaigns, setCampaigns] = useState([])
  const [alerts, setAlerts] = useState([])
  const [showForm, setShowForm] = useState(false)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)

  // TODO Team 6 Sprint 1: fetch campaigns (mock data first, live API Sprint 2)
  // TODO Team 6 Sprint 2: poll every 10s for status changes
  // TODO Team 6 Sprint 3: show alert banner when campaign status changes to PAUSED

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold mb-4">MX Dashboard</h1>
      {alerts.map(alert => <AlertBanner key={alert.id} alert={alert} />)}
      <button
        onClick={() => setShowForm(true)}
        className="mb-4 px-4 py-2 bg-blue-600 text-white rounded"
      >
        New Campaign
      </button>
      {showForm && <CampaignForm onClose={() => setShowForm(false)} />}
      {loading && <p>Loading campaigns...</p>}
      {error && <p className="text-red-600">Error: {error}</p>}
      {!loading && <CampaignList campaigns={campaigns} />}
    </div>
  )
}
```

### pages/CXView.jsx

```jsx
import React, { useState } from 'react'
import { MOCK_CUSTOMERS } from '../constants/testData'
import OfferList from '../components/OfferList'

export default function CXView() {
  const [selectedCustomer, setSelectedCustomer] = useState(null)
  const [cartTotal, setCartTotal] = useState('')
  const [offers, setOffers] = useState(null)
  const [loading, setLoading] = useState(false)

  // TODO Team 6 Sprint 1: implement offer check using eligibilityApi.getOffers()
  // TODO Team 6 Sprint 2: add Redeem button per eligible offer
  // TODO Team 6 Sprint 3: show full eligibility breakdown per offer

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold mb-4">Customer Offer Check</h1>
      <select
        onChange={e => setSelectedCustomer(MOCK_CUSTOMERS.find(c => c.id === e.target.value))}
        className="border p-2 rounded mr-4"
      >
        <option value="">Select customer</option>
        {MOCK_CUSTOMERS.map(c => (
          <option key={c.id} value={c.id}>
            {c.name} ({c.loyaltyTier})
          </option>
        ))}
      </select>
      <input
        type="number"
        placeholder="Cart total $"
        value={cartTotal}
        onChange={e => setCartTotal(e.target.value)}
        className="border p-2 rounded mr-4"
      />
      <button
        className="px-4 py-2 bg-green-600 text-white rounded"
        disabled={!selectedCustomer || loading}
      >
        {loading ? 'Checking...' : 'Check Offers'}
      </button>
      {offers && <OfferList offers={offers} />}
    </div>
  )
}
```

### pages/AnalyticsDashboard.jsx

```jsx
import React, { useState, useEffect } from 'react'
import BurnChart from '../components/BurnChart'
import LiftChart from '../components/LiftChart'

export default function AnalyticsDashboard() {
  const [metrics, setMetrics] = useState([])
  const [loading, setLoading] = useState(true)

  // TODO Team 6 Sprint 2: fetch analytics for all campaigns
  // TODO Team 6 Sprint 2: render burn chart and lift chart

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold mb-4">Analytics</h1>
      {loading && <p>Loading analytics...</p>}
      <BurnChart data={metrics} />
      <LiftChart data={metrics} />
    </div>
  )
}
```

---

## Component Scaffolds

### components/StatusBadge.jsx

```jsx
export default function StatusBadge({ status }) {
  const colours = {
    ACTIVE: 'bg-green-100 text-green-800',
    PAUSED: 'bg-red-100 text-red-800',
    SCHEDULED: 'bg-blue-100 text-blue-800',
    DRAFT: 'bg-gray-100 text-gray-800',
    EXPIRED: 'bg-gray-100 text-gray-500'
  }
  return (
    <span className={`px-2 py-1 rounded text-xs font-medium ${colours[status] || colours.DRAFT}`}>
      {status}
    </span>
  )
}
```

### components/CampaignList.jsx

```jsx
import StatusBadge from './StatusBadge'

export default function CampaignList({ campaigns }) {
  // TODO Team 6 Sprint 1: render campaign table with status badges and burn bars

  const burnColour = (pct) => {
    if (pct >= 95) return 'bg-red-500'
    if (pct >= 80) return 'bg-amber-500'
    return 'bg-green-500'
  }

  return (
    <table className="w-full border-collapse">
      <thead>
        <tr className="bg-gray-50">
          <th className="text-left p-3 border-b">Name</th>
          <th className="text-left p-3 border-b">Status</th>
          <th className="text-left p-3 border-b">Offer Type</th>
          <th className="text-left p-3 border-b">Budget Burn</th>
          <th className="text-left p-3 border-b">Date Range</th>
        </tr>
      </thead>
      <tbody>
        {campaigns.map(c => (
          <tr key={c.id} className="border-b hover:bg-gray-50">
            <td className="p-3">{c.name}</td>
            <td className="p-3"><StatusBadge status={c.status} /></td>
            <td className="p-3">{c.offerType}</td>
            <td className="p-3">
              <div className="flex items-center gap-2">
                <div className="flex-1 bg-gray-200 rounded h-2">
                  <div
                    className={`h-2 rounded ${burnColour(c.budgetBurnPercent)}`}
                    style={{ width: `${Math.min(c.budgetBurnPercent, 100)}%` }}
                  />
                </div>
                <span className="text-sm">{c.budgetBurnPercent?.toFixed(1)}%</span>
              </div>
            </td>
            <td className="p-3 text-sm">{c.dateRange?.startDate} → {c.dateRange?.endDate}</td>
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

### components/AlertBanner.jsx

```jsx
export default function AlertBanner({ alert, onDismiss }) {
  return (
    <div className="bg-red-50 border border-red-200 rounded p-4 mb-4 flex justify-between">
      <div>
        <p className="font-medium text-red-800">{alert.message}</p>
        <p className="text-sm text-red-600">{alert.timestamp}</p>
      </div>
      <button onClick={() => onDismiss?.(alert.id)} className="text-red-400 hover:text-red-600">
        ✕
      </button>
    </div>
  )
}
```

### components/CampaignForm.jsx (Scaffold)

```jsx
import React, { useState } from 'react'

const OFFER_TYPES = ['PCT_OFF', 'AMT_OFF', 'BOGO', 'THRESHOLD']

export default function CampaignForm({ onClose, onSuccess }) {
  const [form, setForm] = useState({
    name: '', offerType: 'PCT_OFF', offerValue: '',
    upcScope: [], dateStart: '', dateEnd: '',
    stackPermission: false, stackLimit: 1,
    vendorId: '', vendorShare: 100, krogerShare: 0
  })
  const [error, setError] = useState(null)
  const [loading, setLoading] = useState(false)

  const vendorShareChange = (val) => {
    setForm(f => ({ ...f, vendorShare: val, krogerShare: 100 - val }))
  }

  const handleSubmit = async (e) => {
    e.preventDefault()
    // TODO Team 6 Sprint 2: call campaignApi.createCampaign() then addFunding()
    setError('Campaign creation not yet implemented')
  }

  return (
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
      <div className="bg-white rounded-lg p-6 w-full max-w-lg">
        <h2 className="text-xl font-bold mb-4">New Campaign</h2>
        {error && <p className="text-red-600 mb-4">{error}</p>}
        <form onSubmit={handleSubmit} className="space-y-4">
          <input placeholder="Campaign name" required
            value={form.name} onChange={e => setForm(f => ({...f, name: e.target.value}))}
            className="w-full border p-2 rounded" />
          <select value={form.offerType}
            onChange={e => setForm(f => ({...f, offerType: e.target.value}))}
            className="w-full border p-2 rounded">
            {OFFER_TYPES.map(t => <option key={t}>{t}</option>)}
          </select>
          <input type="number" placeholder="Offer value"
            value={form.offerValue} onChange={e => setForm(f => ({...f, offerValue: e.target.value}))}
            className="w-full border p-2 rounded" />
          <div className="flex gap-4">
            <input type="date" value={form.dateStart}
              onChange={e => setForm(f => ({...f, dateStart: e.target.value}))}
              className="flex-1 border p-2 rounded" />
            <input type="date" value={form.dateEnd}
              onChange={e => setForm(f => ({...f, dateEnd: e.target.value}))}
              className="flex-1 border p-2 rounded" />
          </div>
          <hr />
          <h3 className="font-medium">Funding</h3>
          <input placeholder="Vendor ID" value={form.vendorId}
            onChange={e => setForm(f => ({...f, vendorId: e.target.value}))}
            className="w-full border p-2 rounded" />
          <div className="flex gap-4">
            <input type="number" min="0" max="100" placeholder="Vendor share %"
              value={form.vendorShare}
              onChange={e => vendorShareChange(Number(e.target.value))}
              className="flex-1 border p-2 rounded" />
            <input type="number" readOnly value={form.krogerShare}
              className="flex-1 border p-2 rounded bg-gray-50" />
          </div>
          {form.vendorShare + form.krogerShare !== 100 && (
            <p className="text-red-600 text-sm">Shares must sum to 100%</p>
          )}
          <div className="flex gap-4 pt-4">
            <button type="button" onClick={onClose}
              className="flex-1 border p-2 rounded">Cancel</button>
            <button type="submit" disabled={loading || form.vendorShare + form.krogerShare !== 100}
              className="flex-1 bg-blue-600 text-white p-2 rounded disabled:opacity-50">
              {loading ? 'Creating...' : 'Create Campaign'}
            </button>
          </div>
        </form>
      </div>
    </div>
  )
}
```

---

## App.jsx

```jsx
import React from 'react'
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom'
import MXDashboard from './pages/MXDashboard'
import CXView from './pages/CXView'
import AnalyticsDashboard from './pages/AnalyticsDashboard'

export default function App() {
  return (
    <BrowserRouter>
      <nav className="bg-gray-900 text-white p-4 flex gap-6">
        <span className="font-bold text-lg">PromotionOS</span>
        <Link to="/mx" className="hover:text-gray-300">MX Dashboard</Link>
        <Link to="/cx" className="hover:text-gray-300">Customer View</Link>
        <Link to="/analytics" className="hover:text-gray-300">Analytics</Link>
      </nav>
      <Routes>
        <Route path="/" element={<MXDashboard />} />
        <Route path="/mx" element={<MXDashboard />} />
        <Route path="/cx" element={<CXView />} />
        <Route path="/analytics" element={<AnalyticsDashboard />} />
      </Routes>
    </BrowserRouter>
  )
}
```

---

## README.md

```markdown
# Frontend Dashboard

React 18 / Vite / Tailwind CSS
Deploy: Vercel (auto-deploy on push to main)

## Documentation
- SDL: [link]
- Architecture: [link]
- Codebase Skill: [link]

## Sprint 1 Note
Campaign list uses mock data from src/constants/testData.js
Switch to live API when Campaign Service is deployed.

## Local Development
\`\`\`bash
cp .env.example .env
# Fill in Railway service URLs
npm install
npm run dev
\`\`\`
```
