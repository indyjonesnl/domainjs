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
const consoleLogs = ref([])
let logIdCounter = 0
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

// Add log message to console
function addLog(message, type = 'info') {
  const id = logIdCounter++
  const log = {
    id,
    message,
    type,
    timestamp: new Date().toLocaleString()
  }

  consoleLogs.value.push(log)
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
    addLog(warnings.join(', '), 'warning')
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

      addLog(
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

// Save to localStorage using individual keys
function saveToStorage() {
  // Clear old data structure (migration)
  localStorage.removeItem('unresolvedDomains')
  localStorage.removeItem('knownServers')
  localStorage.removeItem('resolvedDomains')

  // Clear existing domain and server keys
  const keysToRemove = []
  for (let i = 0; i < localStorage.length; i++) {
    const key = localStorage.key(i)
    if (key && (key.startsWith('domain:') || key.startsWith('server:'))) {
      keysToRemove.push(key)
    }
  }
  keysToRemove.forEach(key => localStorage.removeItem(key))

  // Save unresolved domains individually
  unresolvedDomains.value.forEach(domain => {
    localStorage.setItem(`domain:unresolved:${domain}`, 'true')
  })

  // Save resolved domains individually (group by domain name)
  const resolvedByDomain = {}
  resolvedDomains.value.forEach(resolved => {
    if (!resolvedByDomain[resolved.domain]) {
      resolvedByDomain[resolved.domain] = []
    }
    resolvedByDomain[resolved.domain].push({
      ip: resolved.ip,
      serverName: resolved.serverName,
      resolvedAt: resolved.resolvedAt,
      timestamp: resolved.timestamp
    })
  })

  Object.entries(resolvedByDomain).forEach(([domain, records]) => {
    localStorage.setItem(`domain:resolved:${domain}`, JSON.stringify(records))
  })

  // Save known servers individually
  knownServers.value.forEach(server => {
    localStorage.setItem(`server:${server.name}`, JSON.stringify({ ip: server.ip }))
  })
}

// Load from localStorage using individual keys
function loadFromStorage() {
  unresolvedDomains.value = []
  knownServers.value = []
  resolvedDomains.value = []

  // Iterate through all localStorage keys
  for (let i = 0; i < localStorage.length; i++) {
    const key = localStorage.key(i)
    if (!key) continue

    // Load unresolved domains
    if (key.startsWith('domain:unresolved:')) {
      const domain = key.replace('domain:unresolved:', '')
      unresolvedDomains.value.push(domain)
    }

    // Load resolved domains
    if (key.startsWith('domain:resolved:')) {
      const domain = key.replace('domain:resolved:', '')
      const records = JSON.parse(localStorage.getItem(key) || '[]')

      records.forEach(record => {
        resolvedDomains.value.push({
          domain,
          ip: record.ip,
          serverName: record.serverName,
          resolvedAt: record.resolvedAt,
          timestamp: record.timestamp
        })
      })
    }

    // Load known servers
    if (key.startsWith('server:')) {
      const serverName = key.replace('server:', '')
      const serverData = JSON.parse(localStorage.getItem(key) || '{}')

      if (serverData.ip) {
        knownServers.value.push({
          name: serverName,
          ip: serverData.ip
        })
      }
    }
  }

  // Sort arrays
  unresolvedDomains.value.sort((a, b) => a.localeCompare(b))
  resolvedDomains.value.sort((a, b) => a.domain.localeCompare(b.domain))
}
</script>

<template>
  <div class="app">
    <h1>DNS Domain Resolver</h1>

    <!-- Console Log -->
    <div v-if="consoleLogs.length > 0" class="console-container">
      <div class="console-header">
        <h3>Console</h3>
        <span class="console-count">{{ consoleLogs.length }} {{ consoleLogs.length === 1 ? 'message' : 'messages' }}</span>
      </div>
      <div class="console-logs">
        <div
          v-for="log in consoleLogs"
          :key="log.id"
          :class="['console-log', `console-log-${log.type}`]"
        >
          <span class="console-timestamp">{{ log.timestamp }}</span>
          <span :class="['console-type', `console-type-${log.type}`]">{{ log.type.toUpperCase() }}</span>
          <span class="console-message">{{ log.message }}</span>
        </div>
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

.console-container {
  background: #1e1e1e;
  border: 1px solid #3e3e3e;
  border-radius: 8px;
  margin-bottom: 20px;
  max-height: 400px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.console-header {
  background: #2d2d2d;
  padding: 10px 15px;
  border-bottom: 1px solid #3e3e3e;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.console-header h3 {
  margin: 0;
  color: #42b883;
  font-size: 14px;
  font-weight: 600;
  text-transform: uppercase;
}

.console-count {
  color: #888;
  font-size: 12px;
}

.console-logs {
  overflow-y: auto;
  padding: 10px;
  flex: 1;
}

.console-log {
  display: grid;
  grid-template-columns: auto auto 1fr;
  gap: 12px;
  padding: 8px 10px;
  border-bottom: 1px solid #2d2d2d;
  font-family: 'Courier New', Courier, monospace;
  font-size: 13px;
  line-height: 1.5;
}

.console-log:last-child {
  border-bottom: none;
}

.console-timestamp {
  color: #888;
  font-size: 11px;
  white-space: nowrap;
}

.console-type {
  font-weight: bold;
  font-size: 11px;
  padding: 2px 6px;
  border-radius: 3px;
  white-space: nowrap;
}

.console-type-info {
  color: #4fc3f7;
  background: rgba(79, 195, 247, 0.1);
}

.console-type-warning {
  color: #ffb74d;
  background: rgba(255, 183, 77, 0.1);
}

.console-type-error {
  color: #e57373;
  background: rgba(229, 115, 115, 0.1);
}

.console-message {
  color: #e0e0e0;
  word-break: break-word;
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
