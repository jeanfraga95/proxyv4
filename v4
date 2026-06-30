#!/bin/bash
# ProxyC Installer v4 (Multi-Status Fixed)

set -e

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'
CYAN='\033[0;36m'; BOLD='\033[1m'; NC='\033[0m'

log()  { echo -e "${GREEN}[✔]${NC} $*"; }
warn() { echo -e "${YELLOW}[!]${NC} $*"; }
err()  { echo -e "${RED}[✗]${NC} $*"; exit 1; }
info() { echo -e "${CYAN}[i]${NC} $*"; }
step() { echo -e "\n${CYAN}${BOLD}━━ $* ${NC}"; }

[[ $EUID -ne 0 ]] && err "Execute como root: sudo bash install_proxyc.sh"

INSTALL_DIR="/usr/local/bin"
CONFIG_DIR="/etc/proxyc"
BUILD_DIR="/tmp/proxyc_build_$$"
SERVICE_FILE="/etc/systemd/system/proxyc.service"
GO_VERSION="1.22.4"
PROXYC_VERSION="4.0-multistatus"

echo -e "${CYAN}${BOLD}"
echo "  ╔════════════════════════════════════════╗"
echo "  ║     ProxyC v4 — Multi-Status Fixed     ║"
echo "  ║   SSH Proxy | Multi-Request CDN        ║"
echo "  ╚════════════════════════════════════════╝"
echo -e "${NC}"

step "Limpando instalação anterior"
systemctl stop    proxyc 2>/dev/null && info "Serviço parado"
systemctl disable proxyc 2>/dev/null && info "Serviço desabilitado"
[[ -f "$SERVICE_FILE" ]]            && rm -f "$SERVICE_FILE"            && info "Removido: $SERVICE_FILE"
[[ -f "$INSTALL_DIR/proxyc" ]]      && rm -f "$INSTALL_DIR/proxyc"      && info "Removido: $INSTALL_DIR/proxyc"
[[ -f "$INSTALL_DIR/proxyc-menu" ]] && rm -f "$INSTALL_DIR/proxyc-menu" && info "Removido: $INSTALL_DIR/proxyc-menu"
systemctl daemon-reload 2>/dev/null
log "Limpeza concluída"

step "Verificando Go"
go_ok=0
for gocmd in go /usr/local/go/bin/go /usr/bin/go /usr/local/bin/go; do
    if command -v "$gocmd" &>/dev/null; then
        GOBIN="$(command -v $gocmd 2>/dev/null || echo $gocmd)"
        GOVER=$($GOBIN version 2>/dev/null | grep -oP '\d+\.\d+' | head -1)
        MAJOR=$(echo "$GOVER" | cut -d. -f1)
        MINOR=$(echo "$GOVER" | cut -d. -f2)
        if [[ "${MAJOR:-0}" -ge 1 && "${MINOR:-0}" -ge 20 ]]; then
            log "Go $GOVER: $GOBIN"
            go_ok=1; break
        fi
    fi
done
if [[ $go_ok -eq 0 ]]; then
    info "Instalando Go $GO_VERSION..."
    ARCH=$(uname -m)
    case "$ARCH" in
        x86_64)  GO_ARCH="amd64" ;;
        aarch64) GO_ARCH="arm64" ;;
        armv*)   GO_ARCH="armv6l" ;;
        *)       err "Arquitetura não suportada: $ARCH" ;;
    esac
    GO_TAR="go${GO_VERSION}.linux-${GO_ARCH}.tar.gz"
    command -v curl &>/dev/null || apt-get install -y curl -q
    curl -fL --progress-bar -o "/tmp/${GO_TAR}" "https://dl.google.com/go/${GO_TAR}" || err "Falha ao baixar Go"
    rm -rf /usr/local/go
    tar -C /usr/local -xzf "/tmp/${GO_TAR}" && rm -f "/tmp/${GO_TAR}"
    export PATH="$PATH:/usr/local/go/bin"
    echo 'export PATH=$PATH:/usr/local/go/bin' > /etc/profile.d/go.sh
    GOBIN="/usr/local/go/bin/go"
    log "Go instalado: $($GOBIN version)"
fi

step "Dependências do sistema"
command -v python3 &>/dev/null || { apt-get install -y python3 -q 2>/dev/null || warn "python3 não instalado"; }
command -v ss      &>/dev/null || { apt-get install -y iproute2 -q 2>/dev/null || warn "iproute2 não instalado"; }
log "OK"

step "Compilando ProxyC v4 (Multi-Status)"
mkdir -p "$BUILD_DIR" || err "Não foi possível criar $BUILD_DIR"

# Código corrigido inline (multi-status suportado)
cat > "$BUILD_DIR/main.go" << 'GOEOF'
package main

import (
	"bufio"
	"bytes"
	"encoding/hex"
	"encoding/json"
	"flag"
	"fmt"
	"io"
	"log"
	"net"
	"os"
	"os/signal"
	"strconv"
	"strings"
	"sync"
	"syscall"
	"time"
)

// ==================== CONFIGURAÇÃO ====================

type BackendRule struct {
	Pattern string `json:"pattern"`
	Host    string `json:"host"`
	Port    int    `json:"port"`
}

type Config struct {
	Statuses []string       `json:"statuses"`
	Backends []BackendRule  `json:"backends"`
	Debug    bool           `json:"debug"`
	mu       sync.RWMutex
}

func defaultConfig() *Config {
	return &Config{
		Statuses: []string{"Switching Protocols"},
		Backends: []BackendRule{
			{Pattern: "SSH-", Host: "127.0.0.1", Port: 22},
			{Pattern: "SSH2", Host: "127.0.0.1", Port: 22},
			{Pattern: "", Host: "127.0.0.1", Port: 22},
		},
		Debug: false,
	}
}

var (
	cfg            = defaultConfig()
	statusIdx      int
	statusMu       sync.Mutex
	configFile     string
	debugFlag      bool
	listeners      = make(map[int]net.Listener)
	listenersMu    sync.Mutex
)

// ==================== DEBUG ====================

func isDebug() bool {
	cfg.mu.RLock()
	defer cfg.mu.RUnlock()
	return cfg.Debug || debugFlag
}

func debugLog(format string, args ...interface{}) {
	if isDebug() {
		log.Printf("[DEBUG] "+format, args...)
	}
}

// ==================== STATUS ROUND-ROBIN ====================

func nextStatus() string {
	cfg.mu.RLock()
	statuses := cfg.Statuses
	cfg.mu.RUnlock()

	statusMu.Lock()
	s := statuses[statusIdx%len(statuses)]
	statusIdx++
	statusMu.Unlock()
	return s
}

// ==================== DETECÇÃO DE BACKEND ====================

func detectBackend(data []byte) BackendRule {
	cfg.mu.RLock()
	backends := cfg.Backends
	cfg.mu.RUnlock()

	s := string(data)
	for _, b := range backends {
		if b.Pattern != "" && strings.Contains(s, b.Pattern) {
			debugLog("backend detectado por padrão=%s → %s:%d", b.Pattern, b.Host, b.Port)
			return b
		}
	}
	for _, b := range backends {
		if b.Pattern == "" {
			debugLog("backend fallback → %s:%d", b.Host, b.Port)
			return b
		}
	}
	return BackendRule{Host: "127.0.0.1", Port: 22}
}

// ==================== CONEXÃO ====================

func connectBackend(host string, port int) (net.Conn, error) {
	addr := net.JoinHostPort(host, strconv.Itoa(port))
	conn, err := net.DialTimeout("tcp", addr, 10*time.Second)
	if err != nil {
		return nil, fmt.Errorf("backend %s: %w", addr, err)
	}
	return conn, nil
}

// ==================== CONN READER ====================

type connReader struct {
	conn   net.Conn
	reader *bufio.Reader
}

func newConnReader(conn net.Conn) *connReader {
	return &connReader{
		conn:   conn,
		reader: bufio.NewReaderSize(conn, 65536),
	}
}

func (cr *connReader) peek(n int) []byte {
	cr.conn.SetReadDeadline(time.Now().Add(5 * time.Second))
	data, _ := cr.reader.Peek(n)
	cr.conn.SetReadDeadline(time.Time{})
	return data
}

func (cr *connReader) peekAll(maxWait time.Duration) []byte {
	deadline := time.Now().Add(maxWait)
	var last []byte
	for n := 512; ; n *= 2 {
		if n > 65536 {
			n = 65536
		}
		cr.conn.SetReadDeadline(time.Now().Add(200 * time.Millisecond))
		data, _ := cr.reader.Peek(n)
		cr.conn.SetReadDeadline(time.Time{})
		if len(data) == len(last) || time.Now().After(deadline) {
			last = data
			break
		}
		last = data
		if len(data) < n {
			break
		}
	}
	return last
}

func (cr *connReader) discardUntilBlankLine() []byte {
	cr.conn.SetReadDeadline(time.Now().Add(5 * time.Second))
	defer cr.conn.SetReadDeadline(time.Time{})

	var consumed []byte
	for {
		b, err := cr.reader.ReadByte()
		if err != nil {
			break
		}
		consumed = append(consumed, b)
		l := len(consumed)
		if l >= 4 &&
			consumed[l-4] == '\r' && consumed[l-3] == '\n' &&
			consumed[l-2] == '\r' && consumed[l-1] == '\n' {
			break
		}
		if l >= 2 && consumed[l-2] == '\n' && consumed[l-1] == '\n' {
			break
		}
	}
	return consumed
}

func (cr *connReader) Read(p []byte) (int, error) {
	return cr.reader.Read(p)
}

func (cr *connReader) Write(p []byte) (int, error) {
	return cr.conn.Write(p)
}

// ==================== ANÁLISE HTTP ====================

var httpVerbs = map[string]bool{
	"GET": true, "POST": true, "PUT": true, "DELETE": true,
	"CONNECT": true, "OPTIONS": true, "HEAD": true, "PATCH": true,
	"TRACE": true, "ACL": true, "CHECKIN": true, "CHECKOUT": true,
	"UNLOCK": true, "LOCK": true, "MOVE": true, "COPY": true,
	"REPORT": true, "SEARCH": true, "PROPFIND": true, "PROPPATCH": true,
	"MKCOL": true, "MKACTIVITY": true, "VIEW": true,
}

func firstWord(data []byte) string {
	s := strings.TrimLeft(string(data), "\r\n \t")
	idx := strings.IndexAny(s, " \t\r\n")
	if idx <= 0 {
		return strings.ToUpper(s)
	}
	return strings.ToUpper(s[:idx])
}

func splitHTTPBlocks(data []byte) [][]byte {
	var blocks [][]byte
	for len(data) > 0 {
		idx := bytes.Index(data, []byte("\r\n\r\n"))
		if idx >= 0 {
			blocks = append(blocks, data[:idx+4])
			data = data[idx+4:]
			continue
		}
		idx = bytes.Index(data, []byte("\n\n"))
		if idx >= 0 {
			blocks = append(blocks, data[:idx+2])
			data = data[idx+2:]
			continue
		}
		if len(bytes.TrimSpace(data)) > 0 {
			blocks = append(blocks, data)
		}
		break
	}
	return blocks
}

func parsePayload(data []byte) (httpBlocks int, hasUpgrade bool) {
	blocks := splitHTTPBlocks(data)
	for _, blk := range blocks {
		word := firstWord(blk)
		if !httpVerbs[word] {
			continue
		}
		s := string(blk)
		if !strings.Contains(s, "HTTP/") {
			continue
		}
		httpBlocks++
		if strings.Contains(strings.ToLower(s), "upgrade") {
			hasUpgrade = true
		}
	}
	return
}

// ==================== ENVIO DE RESPOSTAS ====================

func send101(conn net.Conn) {
	status := nextStatus()
	resp := fmt.Sprintf("HTTP/1.1 101 %s\r\n\r\n", status)
	conn.Write([]byte(resp))
	debugLog("→ cliente: 101 %s", status)
}

func send200(conn net.Conn) {
	status := nextStatus()
	resp := fmt.Sprintf("HTTP/1.1 200 OK %s\r\n\r\n", status)
	conn.Write([]byte(resp))
	debugLog("→ cliente: 200 OK %s", status)
}

// ==================== TUNNEL BIDIRECIONAL ====================

func tunnel(client io.ReadWriter, server net.Conn) {
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		n, _ := io.Copy(server, client)
		debugLog("c→s fechou (%d bytes)", n)
		if tc, ok := server.(*net.TCPConn); ok {
			tc.CloseWrite()
		}
	}()

	go func() {
		defer wg.Done()
		n, _ := io.Copy(client.(io.Writer), server)
		debugLog("s→c fechou (%d bytes)", n)
	}()

	wg.Wait()
}

// ==================== HANDLER PRINCIPAL ====================

func handleClient(rawConn net.Conn) {
	defer rawConn.Close()
	remote := rawConn.RemoteAddr().String()

	cr := newConnReader(rawConn)

	peeked := cr.peekAll(3 * time.Second)
	if len(peeked) == 0 {
		debugLog("[%s] sem dados no peek inicial", remote)
		return
	}

	if isDebug() {
		preview := peeked
		if len(preview) > 512 {
			preview = preview[:512]
		}
		debugLog("[%s] peek inicial %d bytes:\n%s\n--- hex ---\n%s",
			remote, len(peeked),
			strings.ReplaceAll(string(preview), "\r", "↵"),
			hex.Dump(preview))
	}

	rawStr := string(peeked)

	if strings.Contains(rawStr, "proxyc:on") || strings.Contains(rawStr, "proxyc: on") {
		send200(rawConn)
		send200(rawConn)
		cr.discardUntilBlankLine()
		goto tunnel_phase
	}

	{
		n, hasUpgrade := parsePayload(peeked)
		if n < 1 {
			n = 1
		}

		log.Printf("[handshake] %s | %d bloco(s) | upgrade=%v", remote, n, hasUpgrade)

		for i := 0; i < n; i++ {
			isLast := i == n-1
			send101(rawConn)
			consumed := cr.discardUntilBlankLine()
			debugLog("[%s] bloco %d/%d consumido: %d bytes", remote, i+1, n, len(consumed))
			if isLast {
				send200(rawConn)
			}
		}
	}

tunnel_phase:
	postPeek := cr.peek(256)
	if isDebug() && len(postPeek) > 0 {
		preview := postPeek
		if len(preview) > 64 {
			preview = preview[:64]
		}
		debugLog("[%s] pós-handshake peek: %q", remote, string(preview))
	}

	backend := detectBackend(postPeek)
	serverConn, err := connectBackend(backend.Host, backend.Port)
	if err != nil {
		log.Printf("[erro] %s → %v", remote, err)
		return
	}
	defer serverConn.Close()

	log.Printf("[tunnel] %s → %s:%d", remote, backend.Host, backend.Port)
	tunnel(cr, serverConn)
}

// ==================== LISTENER MANAGEMENT ====================

func startListener(port int) error {
	listenersMu.Lock()
	if _, exists := listeners[port]; exists {
		listenersMu.Unlock()
		return fmt.Errorf("porta %d já está aberta", port)
	}
	listenersMu.Unlock()

	ln, err := net.Listen("tcp", fmt.Sprintf("[::]::%d", port))
	if err != nil {
		return fmt.Errorf("bind porta %d: %w", port, err)
	}

	listenersMu.Lock()
	listeners[port] = ln
	listenersMu.Unlock()
	log.Printf("[proxy] escutando na porta %d", port)

	go func() {
		for {
			conn, err := ln.Accept()
			if err != nil {
				return
			}
			go handleClient(conn)
		}
	}()

	return nil
}

func stopListener(port int) error {
	listenersMu.Lock()
	ln, exists := listeners[port]
	listenersMu.Unlock()
	if !exists {
		return fmt.Errorf("porta %d não está aberta", port)
	}
	ln.Close()
	listenersMu.Lock()
	delete(listeners, port)
	listenersMu.Unlock()
	log.Printf("[proxy] porta %d fechada", port)
	return nil
}

func listListeners() []int {
	listenersMu.Lock()
	defer listenersMu.Unlock()
	var ports []int
	for p := range listeners {
		ports = append(ports, p)
	}
	return ports
}

// ==================== PERSISTÊNCIA JSON ====================

type PortsFile struct {
	Ports []int `json:"ports"`
}

func savePorts() {
	ports := listListeners()
	data, _ := json.MarshalIndent(PortsFile{Ports: ports}, "", "  ")
	os.WriteFile("/etc/proxyc/ports.json", data, 0644)
}

func loadPorts() []int {
	data, err := os.ReadFile("/etc/proxyc/ports.json")
	if err != nil {
		return nil
	}
	var pf PortsFile
	json.Unmarshal(data, &pf)
	return pf.Ports
}

func loadConfig(path string) {
	data, err := os.ReadFile(path)
	if err != nil {
		return
	}
	var fc struct {
		Statuses []string       `json:"statuses"`
		Backends []BackendRule  `json:"backends"`
		Debug    bool           `json:"debug"`
	}
	if err := json.Unmarshal(data, &fc); err != nil {
		log.Printf("[config] erro: %v", err)
		return
	}
	cfg.mu.Lock()
	if len(fc.Statuses) > 0 {
		cfg.Statuses = fc.Statuses
	}
	if len(fc.Backends) > 0 {
		cfg.Backends = fc.Backends
	}
	cfg.Debug = fc.Debug
	cfg.mu.Unlock()
	log.Printf("[config] recarregada | statuses=%d backends=%d debug=%v",
		len(fc.Statuses), len(fc.Backends), fc.Debug)
}

// ==================== MAIN ====================

func main() {
	var (
		port       = flag.Int("port", 0, "Porta para escutar")
		statusList = flag.String("status-list", "", "Status separados por vírgula")
		upgrade    = flag.String("upgrade", "", "Regras backend: padrão:host:porta,...")
		cfgFile    = flag.String("config", "/etc/proxyc/config.json", "Config JSON")
		debug      = flag.Bool("debug", false, "Ativa logs detalhados de payload")
	)
	flag.Parse()
	configFile = *cfgFile
	debugFlag = *debug

	loadConfig(configFile)

	if *statusList != "" {
		cfg.mu.Lock()
		cfg.Statuses = strings.Split(*statusList, ",")
		cfg.mu.Unlock()
	}
	if *upgrade != "" {
		var rules []BackendRule
		for _, rule := range strings.Split(*upgrade, ",") {
			parts := strings.SplitN(rule, ":", 3)
			if len(parts) == 3 {
				if p, err := strconv.Atoi(parts[2]); err == nil {
					rules = append(rules, BackendRule{Pattern: parts[0], Host: parts[1], Port: p})
				}
			}
		}
		if len(rules) > 0 {
			cfg.mu.Lock()
			cfg.Backends = rules
			cfg.mu.Unlock()
		}
	}

	for _, p := range loadPorts() {
		if err := startListener(p); err != nil {
			log.Printf("[aviso] restaurar porta %d: %v", p, err)
		}
	}

	if *port > 0 {
		if err := startListener(*port); err != nil {
			log.Printf("[aviso] %v", err)
		} else {
			savePorts()
		}
	}

	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGHUP)
	go func() {
		for range sigCh {
			loadConfig(configFile)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
	<-quit

	log.Println("Encerrando...")
	listenersMu.Lock()
	for _, ln := range listeners {
		ln.Close()
	}
	listenersMu.Unlock()
}
GOEOF

[[ -s "$BUILD_DIR/main.go" ]] || err "main.go está vazio"
log "Código fonte: $(wc -l < $BUILD_DIR/main.go) linhas"

cat > "$BUILD_DIR/go.mod" << 'GOMOD'
module proxyc

go 1.21
GOMOD

cd "$BUILD_DIR"
info "Compilando com otimizações..."
BUILD_ERR=$(CGO_ENABLED=0 "$GOBIN" build -ldflags="-s -w" -o "$INSTALL_DIR/proxyc" . 2>&1)
[[ $? -ne 0 ]] && echo "$BUILD_ERR" && err "Compilação falhou"
chmod +x "$INSTALL_DIR/proxyc"
log "Binário: $INSTALL_DIR/proxyc ($(du -sh $INSTALL_DIR/proxyc | cut -f1))"

step "Configuração"
mkdir -p "$CONFIG_DIR"
if [[ ! -f "$CONFIG_DIR/config.json" ]]; then
    cat > "$CONFIG_DIR/config.json" << 'CFGJSON'
{
  "statuses": [
    "Switching Protocols",
    "Web Socket Protocol Handshake",
    "101 Protocol Upgrade"
  ],
  "backends": [
    {"pattern": "SSH-",  "host": "127.0.0.1", "port": 22},
    {"pattern": "SSH2",  "host": "127.0.0.1", "port": 22},
	{"pattern": "SSH",  "host": "127.0.0.1", "port": 22},
    {"pattern": "",      "host": "127.0.0.1", "port": 22}
  ],
  "debug": false
}
CFGJSON
    log "Config criada: $CONFIG_DIR/config.json"
else
    warn "Config existente mantida"
fi
[[ ! -f "$CONFIG_DIR/ports.json" ]] && echo '{"ports":[]}' > "$CONFIG_DIR/ports.json"
chmod 644 "$CONFIG_DIR"/*.json

step "Serviço systemd"
cat > "$SERVICE_FILE" << SVCEOF
[Unit]
Description=ProxyC SSH Proxy - Multi-Status v4
After=network.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=$INSTALL_DIR/proxyc --config $CONFIG_DIR/config.json
ExecReload=/bin/kill -HUP \$MAINPID
Restart=always
RestartSec=3
KillMode=process
LimitNOFILE=65536

StandardOutput=journal
StandardError=journal
SyslogIdentifier=proxyc

[Install]
WantedBy=multi-user.target
SVCEOF

systemctl daemon-reload
systemctl enable proxyc 2>/dev/null && log "Habilitado no boot"
systemctl start proxyc 2>/dev/null
sleep 2
if systemctl is-active --quiet proxyc 2>/dev/null; then
    log "Serviço ATIVO (Multi-Status v4)"
else
    warn "Systemd falhou. Iniciando direto..."
    "$INSTALL_DIR/proxyc" --config "$CONFIG_DIR/config.json" &
    sleep 1
    pgrep -x proxyc &>/dev/null && log "Rodando em background (PID: $(pgrep -x proxyc))"
fi

rm -rf "$BUILD_DIR"

echo ""
echo -e "${CYAN}${BOLD}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo -e "${BOLD}  ProxyC v4 Multi-Status (CORRIGIDO) instalado!${NC}"
echo -e "${CYAN}${BOLD}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo ""
echo -e "  ${GREEN}✔ Suporte a múltiplas requisições HTTP (CloudFront CDN)${NC}"
echo -e "  ${GREEN}✔ Respostas 101 por bloco + 200 final${NC}"
echo -e "  ${GREEN}✔ Buffer de 64KB para payload completo${NC}"
echo ""
echo -e "  Uso:"
echo -e "    $INSTALL_DIR/proxyc --port 2086"
echo -e "    $INSTALL_DIR/proxyc --port 2086 --debug"
echo ""
echo -e "  Status:"
echo -e "    systemctl status proxyc"
echo -e "    journalctl -u proxyc -f"
echo ""
echo -e "  Teste de multi-status:"
echo -e "    telnet 127.0.0.1 2086"
echo -e "    # Cole 3 requisições GET concatenadas"
echo ""
