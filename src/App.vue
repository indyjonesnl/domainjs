<script setup>
import { ref, computed, onMounted } from 'vue'

// State
const unresolvedDomains = ref([])
const knownServers = ref([])
const resolvedDomains = ref([])
const newDomain = ref('')
const newServerName = ref('')
const newServerIP = ref('')
const isResolving = ref(false)
const isRetryingAll = ref(false)
const warningMessage = ref('')
const warningTimeout = ref(null)
const toasts = ref([])
let toastIdCounter = 0
const recentlyRetriedDomains = ref(new Set())

// Load data from localStorage on mount
onMounted(() => {
  loadFromStorage()
})

// DNS resolution using Cloudflare DNS-over-HTTPS
async function resolveDomain(domain) {
  try {
    const response = await fetch(
      `https://cloudflare-dns.com/dns-query?name=${domain}&type=A`,
      {
        headers: {
          'Accept': 'application/dns-json'
        }
      }
    )

    if (!response.ok) {
      throw new Error('DNS resolution failed')
    }

    const data = await response.json()

    if (data.Answer && data.Answer.length > 0) {
      // Filter for A records (IPv4 addresses)
      const aRecords = data.Answer.filter(record => record.type === 1)
      if (aRecords.length > 0) {
        return aRecords.map(record => record.data)
      }
    }

    return null
  } catch (error) {
    console.error(`Failed to resolve ${domain}:`, error)
    return null
  }
}

// Show warning message
function showWarning(message) {
  warningMessage.value = message

  // Clear existing timeout
  if (warningTimeout.value) {
    clearTimeout(warningTimeout.value)
  }

  // Auto-dismiss after 4 seconds
  warningTimeout.value = setTimeout(() => {
    warningMessage.value = ''
  }, 4000)
}

// Show toast notification
function showToast(message, type = 'warning') {
  const id = toastIdCounter++
  const toast = {
    id,
    message,
    type
  }

  toasts.value.push(toast)

  // Auto-dismiss after 6 seconds
  setTimeout(() => {
    removeToast(id)
  }, 6000)
}

// Remove toast by ID
function removeToast(id) {
  toasts.value = toasts.value.filter(toast => toast.id !== id)
}

// Add a new domain to the unresolved list
function addDomain() {
  const input = newDomain.value.trim()

  if (!input) {
    return
  }

  // Split by comma and trim each domain
  const domains = input.split(',').map(d => d.trim()).filter(d => d !== '')

  if (domains.length === 0) {
    return
  }

  const addedDomains = []
  const warnings = []

  for (const domain of domains) {
    // Check if domain already exists in unresolved list
    if (unresolvedDomains.value.includes(domain)) {
      warnings.push(`"${domain}" is already in the unresolved list`)
      continue
    }

    // Check if domain already exists in resolved list
    const alreadyResolved = resolvedDomains.value.some(resolved => resolved.domain === domain)
    if (alreadyResolved) {
      warnings.push(`"${domain}" has already been resolved`)
      continue
    }

    // Add domain
    unresolvedDomains.value.push(domain)
    addedDomains.push(domain)
  }

  // Sort alphabetically if any domains were added
  if (addedDomains.length > 0) {
    unresolvedDomains.value.sort((a, b) => a.localeCompare(b))
    saveToStorage()
  }

  // Clear input
  newDomain.value = ''

  // Show warnings if any
  if (warnings.length > 0) {
    showWarning(warnings.join(', '))
  }
}

// Add a new known server
function addKnownServer() {
  const name = newServerName.value.trim()
  const ip = newServerIP.value.trim()

  if (name && ip) {
    knownServers.value.push({ name, ip })
    newServerName.value = ''
    newServerIP.value = ''
    saveToStorage()
  }
}

// Remove a known server
function removeServer(index) {
  knownServers.value.splice(index, 1)
  saveToStorage()
}

// Resolve all unresolved domains
async function resolveAllDomains() {
  if (unresolvedDomains.value.length === 0 || isResolving.value) {
    return
  }

  isResolving.value = true

  const domainsToResolve = [...unresolvedDomains.value]

  for (const domain of domainsToResolve) {
    const ips = await resolveDomain(domain)

    if (ips && ips.length > 0) {
      // Remove from unresolved
      unresolvedDomains.value = unresolvedDomains.value.filter(d => d !== domain)

      // Add to resolved with timestamp
      for (const ip of ips) {
        const matchedServer = knownServers.value.find(server => server.ip === ip)

        resolvedDomains.value.push({
          domain,
          ip,
          serverName: matchedServer ? matchedServer.name : null,
          resolvedAt: new Date().toLocaleString(),
          timestamp: Date.now()
        })
      }

      // Sort resolved domains alphabetically by domain name
      resolvedDomains.value.sort((a, b) => a.domain.localeCompare(b.domain))

      saveToStorage()
    }
  }

  isResolving.value = false
}

// Retry resolving a single domain
async function retryResolveDomain(domain) {
  const ips = await resolveDomain(domain)

  if (ips && ips.length > 0) {
    // Get existing entries for this domain
    const existingEntries = resolvedDomains.value.filter(rd => rd.domain === domain)
    const oldIPs = existingEntries.map(entry => entry.ip)

    // Check if IPs changed
    const newIPs = ips.filter(ip => !oldIPs.includes(ip))
    const removedIPs = oldIPs.filter(ip => !ips.includes(ip))

    // Remove old entries for this domain
    resolvedDomains.value = resolvedDomains.value.filter(rd => rd.domain !== domain)

    // Add updated entries
    for (const ip of ips) {
      const matchedServer = knownServers.value.find(server => server.ip === ip)

      resolvedDomains.value.push({
        domain,
        ip,
        serverName: matchedServer ? matchedServer.name : null,
        resolvedAt: new Date().toLocaleString(),
        timestamp: Date.now()
      })
    }

    // Show toast if IPs changed
    if (newIPs.length > 0 || removedIPs.length > 0) {
      const oldServerNames = existingEntries.map(e => e.serverName || 'Unmatched').join(', ')
      const newServerNames = ips.map(ip => {
        const server = knownServers.value.find(s => s.ip === ip)
        return server ? server.name : 'Unmatched'
      }).join(', ')

      showToast(
        `⚠️ IP Address Changed! "${domain}" resolved from ${oldServerNames} (${oldIPs.join(', ')}) to ${newServerNames} (${ips.join(', ')})`,
        'warning'
      )
    }

    // Mark domain as recently retried for flash effect
    recentlyRetriedDomains.value.add(domain)
    setTimeout(() => {
      recentlyRetriedDomains.value.delete(domain)
    }, 3000)

    // Sort resolved domains alphabetically by domain name
    resolvedDomains.value.sort((a, b) => a.domain.localeCompare(b.domain))

    saveToStorage()
  }
}

// Retry all resolved domains with delay
async function retryAllResolvedDomains() {
  if (resolvedDomains.value.length === 0 || isRetryingAll.value) {
    return
  }

  isRetryingAll.value = true

  // Get unique domain names from resolved domains
  const uniqueDomains = [...new Set(resolvedDomains.value.map(rd => rd.domain))]

  for (let i = 0; i < uniqueDomains.length; i++) {
    const domain = uniqueDomains[i]
    await retryResolveDomain(domain)

    // Add 100ms delay between retries (except for the last one)
    if (i < uniqueDomains.length - 1) {
      await new Promise(resolve => setTimeout(resolve, 100))
    }
  }

  isRetryingAll.value = false
}

// Remove a domain from unresolved list
function removeDomain(domain) {
  unresolvedDomains.value = unresolvedDomains.value.filter(d => d !== domain)
  saveToStorage()
}

// Remove a resolved domain
function removeResolvedDomain(index) {
  resolvedDomains.value.splice(index, 1)
  saveToStorage()
}

// Remove all unmatched domains
function removeAllUnmatched() {
  resolvedDomains.value = resolvedDomains.value.filter(resolved => resolved.serverName !== null)
  saveToStorage()
}

// Grouped domains by server
const groupedDomains = computed(() => {
  const groups = {
    unmatched: []
  }

  // Create groups for each known server
  knownServers.value.forEach(server => {
    groups[server.name] = []
  })

  // Group resolved domains
  resolvedDomains.value.forEach(resolved => {
    if (resolved.serverName) {
      if (!groups[resolved.serverName]) {
        groups[resolved.serverName] = []
      }
      groups[resolved.serverName].push(resolved)
    } else {
      groups.unmatched.push(resolved)
    }
  })

  return groups
})

// Save to localStorage
function saveToStorage() {
  localStorage.setItem('unresolvedDomains', JSON.stringify(unresolvedDomains.value))
  localStorage.setItem('knownServers', JSON.stringify(knownServers.value))
  localStorage.setItem('resolvedDomains', JSON.stringify(resolvedDomains.value))
}

// Load from localStorage
function loadFromStorage() {
  const savedUnresolved = localStorage.getItem('unresolvedDomains')
  const savedServers = localStorage.getItem('knownServers')
  const savedResolved = localStorage.getItem('resolvedDomains')

  if (savedUnresolved) {
    unresolvedDomains.value = JSON.parse(savedUnresolved)
    // Sort alphabetically
    unresolvedDomains.value.sort((a, b) => a.localeCompare(b))
  }

  if (savedServers) {
    knownServers.value = JSON.parse(savedServers)
  }

  if (savedResolved) {
    resolvedDomains.value = JSON.parse(savedResolved)
    // Sort alphabetically by domain name
    resolvedDomains.value.sort((a, b) => a.domain.localeCompare(b.domain))
  }
}
</script>

<template>
  <div class="app">
    <h1>DNS Domain Resolver</h1>

    <!-- Warning Message -->
    <div v-if="warningMessage" class="warning-message">
      ⚠️ {{ warningMessage }}
    </div>

    <!-- Toast Notifications -->
    <div class="toast-container">
      <div
        v-for="toast in toasts"
        :key="toast.id"
        :class="['toast', `toast-${toast.type}`]"
      >
        <span class="toast-message">{{ toast.message }}</span>
        <button class="toast-close" @click="removeToast(toast.id)">×</button>
      </div>
    </div>

    <!-- Controls Section -->
    <div class="controls">
      <div class="control-group">
        <h3>Add Known Server</h3>
        <div class="input-row">
          <input
            v-model="newServerName"
            placeholder="Server name"
            @keyup.enter="addKnownServer"
          />
          <input
            v-model="newServerIP"
            placeholder="IP address"
            @keyup.enter="addKnownServer"
          />
          <button @click="addKnownServer">Add Server</button>
        </div>
      </div>

      <div class="control-group">
        <h3>Add Domain to Resolve</h3>
        <div class="input-row">
          <input
            v-model="newDomain"
            placeholder="example.com"
            @keyup.enter="addDomain"
          />
          <button @click="addDomain">Add Domain</button>
        </div>
      </div>

      <div class="control-group">
        <div class="button-row">
          <button
            class="resolve-btn"
            @click="resolveAllDomains"
            :disabled="unresolvedDomains.length === 0 || isResolving || isRetryingAll"
          >
            {{ isResolving ? 'Resolving...' : 'Resolve All Domains' }}
          </button>
          <button
            class="retry-all-btn"
            @click="retryAllResolvedDomains"
            :disabled="resolvedDomains.length === 0 || isResolving || isRetryingAll"
          >
            {{ isRetryingAll ? 'Retrying...' : 'Retry All Resolved' }}
          </button>
        </div>
      </div>
    </div>

    <!-- Top Section: Known Servers (Left) + Unresolved/Unmatched (Right) -->
    <div class="top-section">
      <!-- Left: Known Servers -->
      <div class="section known-servers-section">
        <h2>Known Servers ({{ knownServers.length }})</h2>
        <div class="server-items">
          <div v-if="knownServers.length === 0" class="empty-state">
            No known servers
          </div>
          <div v-for="(server, index) in knownServers" :key="index" class="server-item">
            <span><strong>{{ server.name }}</strong>: {{ server.ip }}</span>
            <button class="remove-btn" @click="removeServer(index)">Remove</button>
          </div>
        </div>
      </div>

      <!-- Right: Unresolved + Unmatched -->
      <div class="right-column">
        <!-- Unresolved Domains -->
        <div class="section">
          <h2>Unresolved Domains ({{ unresolvedDomains.length }})</h2>
          <div class="domain-list">
            <div v-if="unresolvedDomains.length === 0" class="empty-state">
              No unresolved domains
            </div>
            <div
              v-for="domain in unresolvedDomains"
              :key="domain"
              class="domain-item"
            >
              <span class="domain-name">{{ domain }}</span>
              <button class="remove-btn" @click="removeDomain(domain)">Remove</button>
            </div>
          </div>
        </div>

        <!-- Visual Divider -->
        <div class="column-divider"></div>

        <!-- Unmatched Domains -->
        <div class="section">
          <div class="section-header">
            <h2 class="unmatched-header">Unmatched ({{ groupedDomains.unmatched.length }})</h2>
            <button
              class="delete-all-btn"
              @click="removeAllUnmatched"
              :disabled="groupedDomains.unmatched.length === 0"
            >
              Delete All
            </button>
          </div>
          <div class="resolved-list">
            <div v-if="groupedDomains.unmatched.length === 0" class="empty-state">
              No unmatched domains
            </div>
            <div
              v-for="(resolved, index) in groupedDomains.unmatched"
              :key="`unmatched-${index}`"
              :class="['resolved-item', { 'flash-retry': recentlyRetriedDomains.has(resolved.domain) }]"
            >
              <div class="resolved-info">
                <strong>{{ resolved.domain }}</strong>
                <span class="ip">IP: {{ resolved.ip }}</span>
                <span class="timestamp">Resolved: {{ resolved.resolvedAt }}</span>
              </div>
              <div class="resolved-actions">
                <button @click="retryResolveDomain(resolved.domain)">Retry</button>
                <button class="remove-btn" @click="removeResolvedDomain(resolvedDomains.indexOf(resolved))">Remove</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Bottom Section: Matched Domains by Server (Full Width Horizontal Grid) -->
    <div class="bottom-section">
      <h2 class="servers-title">Matched Domains by Server</h2>
      <div class="servers-grid">
        <div
          v-for="server in knownServers"
          :key="server.name"
          class="server-column"
        >
          <h3>{{ server.name }} ({{ server.ip }})</h3>
          <div class="server-count">{{ groupedDomains[server.name]?.length || 0 }} domains</div>
          <div class="resolved-list">
            <div v-if="!groupedDomains[server.name] || groupedDomains[server.name].length === 0" class="empty-state">
              No domains
            </div>
            <div
              v-for="(resolved, index) in groupedDomains[server.name]"
              :key="`${server.name}-${index}`"
              :class="['resolved-item', { 'flash-retry': recentlyRetriedDomains.has(resolved.domain) }]"
            >
              <div class="resolved-info">
                <strong>{{ resolved.domain }}</strong>
                <span class="ip">IP: {{ resolved.ip }}</span>
                <span class="timestamp">Resolved: {{ resolved.resolvedAt }}</span>
              </div>
              <div class="resolved-actions">
                <button @click="retryResolveDomain(resolved.domain)">Retry</button>
                <button class="remove-btn" @click="removeResolvedDomain(resolvedDomains.indexOf(resolved))">Remove</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app {
  max-width: 2400px;
  margin: 0 auto;
  padding: 20px;
}

h1 {
  text-align: center;
  color: #42b883;
  margin-bottom: 30px;
}

.warning-message {
  background: #fff3cd;
  border: 2px solid #ffc107;
  color: #856404;
  padding: 15px 20px;
  border-radius: 8px;
  margin-bottom: 20px;
  text-align: center;
  font-weight: 500;
  animation: slideDown 0.3s ease-out;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.toast-container {
  position: fixed;
  top: 20px;
  right: 20px;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-width: 500px;
}

.toast {
  background: white;
  border-radius: 8px;
  padding: 16px 20px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  display: flex;
  align-items: flex-start;
  gap: 12px;
  animation: slideInRight 0.3s ease-out;
  border-left: 4px solid #ffc107;
}

.toast-warning {
  border-left-color: #ff9800;
  background: #fff8e1;
}

.toast-message {
  flex: 1;
  color: #333;
  font-size: 14px;
  line-height: 1.5;
  word-break: break-word;
}

.toast-close {
  background: none;
  border: none;
  color: #999;
  font-size: 24px;
  line-height: 1;
  cursor: pointer;
  padding: 0;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: color 0.2s;
}

.toast-close:hover {
  color: #333;
  background: transparent;
}

@keyframes slideInRight {
  from {
    opacity: 0;
    transform: translateX(100px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.controls {
  background: #f5f5f5;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 30px;
}

.control-group {
  margin-bottom: 20px;
}

.control-group:last-child {
  margin-bottom: 0;
}

.control-group h3 {
  margin-top: 0;
  margin-bottom: 10px;
  color: #333;
}

.input-row {
  display: flex;
  gap: 10px;
}

input {
  flex: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

button {
  padding: 10px 20px;
  background: #42b883;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.3s;
}

button:hover {
  background: #35a372;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.button-row {
  display: flex;
  gap: 10px;
}

.resolve-btn {
  flex: 1;
  padding: 15px;
  font-size: 16px;
  font-weight: bold;
}

.retry-all-btn {
  flex: 1;
  padding: 15px;
  font-size: 16px;
  font-weight: bold;
  background: #ff9800;
  border: 2px solid #f57c00;
}

.retry-all-btn:hover:not(:disabled) {
  background: #f57c00;
}

.retry-all-btn:disabled {
  background: #ccc;
  border-color: #ccc;
}

.server-items {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.server-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  background: #f9f9f9;
  border-radius: 4px;
  border: 1px solid #e0e0e0;
}

.top-section {
  display: flex;
  gap: 20px;
  align-items: flex-start;
  margin-bottom: 30px;
}

.known-servers-section {
  flex: 0 0 700px;
}

.right-column {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0;
  min-width: 0;
}

.section {
  background: white;
  padding: 20px;
  border-radius: 8px;
  border: 1px solid #e0e0e0;
}

.section h2 {
  margin-top: 0;
  color: #333;
  border-bottom: 2px solid #42b883;
  padding-bottom: 10px;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
  padding-bottom: 10px;
  border-bottom: 2px solid #e67e22;
  margin-bottom: 15px;
}

.section-header h2 {
  margin-bottom: 0;
  padding-bottom: 0;
  border-bottom: none;
  flex: 1;
}

.delete-all-btn {
  padding: 8px 16px;
  font-size: 13px;
  background: #e74c3c;
  white-space: nowrap;
}

.delete-all-btn:hover:not(:disabled) {
  background: #c0392b;
}

.delete-all-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.column-divider {
  height: 30px;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.column-divider::before {
  content: '';
  position: absolute;
  left: 20px;
  right: 20px;
  height: 2px;
  background: linear-gradient(to right, transparent, #ddd, transparent);
}

.bottom-section {
  width: 100%;
}

.servers-title {
  margin-top: 0;
  margin-bottom: 20px;
  color: #333;
  font-size: 24px;
  text-align: center;
}

.servers-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 20px;
  align-items: start;
}

.server-column {
  background: white;
  padding: 20px;
  border-radius: 8px;
  border: 1px solid #e0e0e0;
  display: flex;
  flex-direction: column;
  min-height: 200px;
}

.server-column h3 {
  margin-top: 0;
  margin-bottom: 5px;
  color: #333;
  font-size: 16px;
  border-bottom: 2px solid #42b883;
  padding-bottom: 8px;
}

.server-count {
  font-size: 12px;
  color: #666;
  margin-bottom: 15px;
  font-weight: 500;
}

.domain-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.domain-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px;
  background: #f9f9f9;
  border-radius: 4px;
  border: 1px solid #e0e0e0;
}

.domain-name {
  font-weight: 500;
  color: #333;
}

.unmatched-header {
  color: #e67e22 !important;
  border-bottom-color: #e67e22 !important;
}

.resolved-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.resolved-item {
  padding: 12px;
  background: #f9f9f9;
  border-radius: 4px;
  border: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: background-color 3s ease-out, border-color 3s ease-out;
}

.resolved-item.flash-retry {
  background: #ffe0b2;
  border-color: #ff9800;
  animation: flashOrange 3s ease-out;
}

@keyframes flashOrange {
  0% {
    background: #ff9800;
    border-color: #f57c00;
  }
  100% {
    background: #f9f9f9;
    border-color: #e0e0e0;
  }
}

.resolved-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.resolved-info strong {
  color: #333;
}

.ip {
  color: #666;
  font-size: 14px;
}

.timestamp {
  color: #999;
  font-size: 12px;
}

.resolved-actions {
  display: flex;
  gap: 8px;
}

.resolved-actions button {
  padding: 6px 12px;
  font-size: 12px;
}

.remove-btn {
  background: #e74c3c;
}

.remove-btn:hover {
  background: #c0392b;
}

.empty-state {
  padding: 20px;
  text-align: center;
  color: #999;
  font-style: italic;
}

@media (max-width: 1200px) {
  .top-section {
    flex-direction: column;
  }

  .known-servers-section {
    flex: 1;
    width: 100%;
  }

  .right-column {
    flex: 1;
    width: 100%;
  }

  .servers-grid {
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  }
}

@media (max-width: 768px) {
  .input-row {
    flex-direction: column;
  }

  .button-row {
    flex-direction: column;
  }

  .servers-grid {
    grid-template-columns: 1fr;
  }
}
</style>
