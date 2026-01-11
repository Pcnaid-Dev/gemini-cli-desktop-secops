# Security Recommendations & Implementation Guide

## Critical Security Fixes Required

This document provides actionable recommendations to address the security vulnerabilities identified in the security review.

---

## 1. Web Server Network Exposure (CRITICAL)

### Current Issue
```rust
// crates/server/src/main.rs
rocket::custom(
    rocket::Config::figment()
        .merge(("port", 1858))
        .merge(("address", "0.0.0.0")),  // ❌ Exposes to network
)
```

### Recommended Fix

**Option A: Localhost Only (Recommended Default)**
```rust
rocket::custom(
    rocket::Config::figment()
        .merge(("port", 1858))
        .merge(("address", "127.0.0.1")),  // ✅ Localhost only
)
```

**Option B: Configurable with Warning**
```rust
let bind_address = std::env::var("GEMINI_BIND_ADDRESS")
    .unwrap_or_else(|_| "127.0.0.1".to_string());

if bind_address != "127.0.0.1" {
    eprintln!("⚠️  WARNING: Server binding to {} - accessible from network!", bind_address);
    eprintln!("⚠️  This exposes your application to unauthorized access.");
    eprintln!("⚠️  Ensure proper firewall rules and authentication are in place.");
}

rocket::custom(
    rocket::Config::figment()
        .merge(("port", 1858))
        .merge(("address", bind_address)),
)
```

### Implementation Steps

1. Update `crates/server/src/main.rs`
2. Add environment variable support
3. Update documentation with security warnings
4. Add startup banner with security status

---

## 2. API Authentication (CRITICAL)

### Recommended Implementation

**Step 1: Generate API Token**

Add to `crates/server/src/main.rs`:

```rust
use std::sync::Arc;
use rocket::http::Status;
use rocket::request::{self, Request, FromRequest};

// Generate or load API token
fn get_or_create_api_token() -> String {
    use sha2::{Sha256, Digest};
    
    let token_file = dirs::home_dir()
        .unwrap()
        .join(".gemini-cli-desktop")
        .join("api_token");
    
    if token_file.exists() {
        std::fs::read_to_string(&token_file).unwrap().trim().to_string()
    } else {
        // Generate random token
        let random_bytes: Vec<u8> = (0..32).map(|_| rand::random::<u8>()).collect();
        let mut hasher = Sha256::new();
        hasher.update(&random_bytes);
        let token = format!("{:x}", hasher.finalize());
        
        // Save token
        std::fs::create_dir_all(token_file.parent().unwrap()).unwrap();
        std::fs::write(&token_file, &token).unwrap();
        
        println!("🔐 Generated new API token: {}", token);
        println!("🔐 Token saved to: {}", token_file.display());
        
        token
    }
}

// API Token guard
pub struct ApiToken;

#[rocket::async_trait]
impl<'r> FromRequest<'r> for ApiToken {
    type Error = ();

    async fn from_request(request: &'r Request<'_>) -> request::Outcome<Self, Self::Error> {
        let expected_token = request.rocket().state::<Arc<String>>().unwrap();
        
        match request.headers().get_one("Authorization") {
            Some(token) => {
                let token = token.trim_start_matches("Bearer ");
                if token == expected_token.as_str() {
                    request::Outcome::Success(ApiToken)
                } else {
                    request::Outcome::Error((Status::Unauthorized, ()))
                }
            }
            None => request::Outcome::Error((Status::Unauthorized, ())),
        }
    }
}
```

**Step 2: Protect Routes**

```rust
// Add ApiToken guard to sensitive routes
#[post("/send-message", data = "<request>")]
async fn send_message(
    _token: ApiToken,  // ✅ Requires authentication
    request: Json<SendMessageRequest>, 
    state: &State<AppState>
) -> AppResult<()> {
    // ... existing code
}

#[post("/execute-command", data = "<request>")]
async fn execute_confirmed_command(
    _token: ApiToken,  // ✅ Requires authentication
    request: Json<ExecuteCommandRequest>,
    state: &State<AppState>,
) -> AppResult<Json<String>> {
    // ... existing code
}
```

**Step 3: Update Main Function**

```rust
#[rocket::launch]
fn rocket() -> _ {
    let api_token = Arc::new(get_or_create_api_token());
    
    // ... existing setup ...
    
    rocket::custom(config)
        .manage(app_state)
        .manage(api_token)  // ✅ Add token to state
        .mount("/", routes![index])
        .mount("/api", routes![...])
}
```

---

## 3. Input Validation & Path Traversal Prevention

### Path Traversal Protection

Add to `crates/backend/src/filesystem/mod.rs`:

```rust
use std::path::{Path, PathBuf};

/// Validates that a path doesn't contain path traversal attempts
pub fn validate_safe_path(path: &str) -> Result<PathBuf> {
    let path = Path::new(path);
    
    // Check for path traversal patterns
    let path_str = path.to_string_lossy();
    if path_str.contains("..") || path_str.contains("//") {
        anyhow::bail!("Path traversal detected: {}", path_str);
    }
    
    // Canonicalize to get absolute path
    let canonical = path.canonicalize()
        .context("Failed to canonicalize path")?;
    
    // Ensure it's within allowed directories (e.g., user home)
    let home = std::env::var("HOME")
        .or_else(|_| std::env::var("USERPROFILE"))?;
    let home_path = Path::new(&home).canonicalize()?;
    
    if !canonical.starts_with(&home_path) {
        anyhow::bail!("Access denied: path outside allowed directory");
    }
    
    Ok(canonical)
}
```

### Command Validation

Add to `crates/backend/src/lib.rs`:

```rust
/// Validates command for suspicious patterns
fn validate_command(command: &str) -> Result<()> {
    // Blocked patterns
    let blocked_patterns = [
        "rm -rf /",
        "format ",
        "mkfs",
        "dd if=",
        ":(){:|:&};:",  // Fork bomb
        "curl | sh",
        "wget | sh",
    ];
    
    let command_lower = command.to_lowercase();
    for pattern in &blocked_patterns {
        if command_lower.contains(pattern) {
            anyhow::bail!("Blocked command pattern detected: {}", pattern);
        }
    }
    
    // Log command for audit
    log_command_execution(command);
    
    Ok(())
}

fn log_command_execution(command: &str) {
    let log_dir = dirs::home_dir()
        .unwrap()
        .join(".gemini-cli-desktop")
        .join("audit");
    std::fs::create_dir_all(&log_dir).ok();
    
    let log_file = log_dir.join("command_audit.log");
    let timestamp = chrono::Utc::now().to_rfc3339();
    let entry = format!("[{}] Command: {}\n", timestamp, command);
    
    use std::fs::OpenOptions;
    use std::io::Write;
    
    if let Ok(mut file) = OpenOptions::new()
        .create(true)
        .append(true)
        .open(log_file)
    {
        let _ = file.write_all(entry.as_bytes());
    }
}
```

---

## 4. Rate Limiting

Add to `crates/server/src/main.rs`:

```rust
use std::collections::HashMap;
use std::time::{Duration, Instant};
use rocket::request::{self, FromRequest, Request};

pub struct RateLimiter {
    requests: Arc<Mutex<HashMap<String, Vec<Instant>>>>,
    max_requests: usize,
    window: Duration,
}

impl RateLimiter {
    pub fn new(max_requests: usize, window_secs: u64) -> Self {
        Self {
            requests: Arc::new(Mutex::new(HashMap::new())),
            max_requests,
            window: Duration::from_secs(window_secs),
        }
    }
    
    pub fn check(&self, ip: &str) -> bool {
        let mut requests = self.requests.lock().unwrap();
        let now = Instant::now();
        
        let ip_requests = requests.entry(ip.to_string()).or_insert_with(Vec::new);
        
        // Remove old requests outside the window
        ip_requests.retain(|&time| now.duration_since(time) < self.window);
        
        if ip_requests.len() >= self.max_requests {
            false
        } else {
            ip_requests.push(now);
            true
        }
    }
}

pub struct RateLimit;

#[rocket::async_trait]
impl<'r> FromRequest<'r> for RateLimit {
    type Error = ();

    async fn from_request(request: &'r Request<'_>) -> request::Outcome<Self, Self::Error> {
        let limiter = request.rocket().state::<RateLimiter>().unwrap();
        let ip = request.client_ip()
            .map(|ip| ip.to_string())
            .unwrap_or_else(|| "unknown".to_string());
        
        if limiter.check(&ip) {
            request::Outcome::Success(RateLimit)
        } else {
            request::Outcome::Error((Status::TooManyRequests, ()))
        }
    }
}
```

---

## 5. HTTPS/TLS Support

### Add TLS Configuration

Update Cargo.toml for server:
```toml
[dependencies]
rocket = { version = "0.5.1", features = ["json", "tls"] }
```

Update server configuration:
```rust
use rocket::config::{TlsConfig, CipherSuite};

let tls_config = TlsConfig::from_paths(
    "/path/to/cert.pem",
    "/path/to/key.pem"
).with_ciphers([
    CipherSuite::TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
    CipherSuite::TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
]);

rocket::custom(
    rocket::Config::figment()
        .merge(("port", 1858))
        .merge(("address", "127.0.0.1"))
        .merge(("tls", tls_config))
)
```

---

## 6. Security Headers

Add to routes:
```rust
use rocket::fairing::{Fairing, Info, Kind};
use rocket::{Request, Response};

pub struct SecurityHeaders;

#[rocket::async_trait]
impl Fairing for SecurityHeaders {
    fn info(&self) -> Info {
        Info {
            name: "Security Headers",
            kind: Kind::Response,
        }
    }

    async fn on_response<'r>(&self, _request: &'r Request<'_>, response: &mut Response<'r>) {
        response.set_raw_header("X-Content-Type-Options", "nosniff");
        response.set_raw_header("X-Frame-Options", "DENY");
        response.set_raw_header("X-XSS-Protection", "1; mode=block");
        response.set_raw_header(
            "Content-Security-Policy",
            "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
        );
        response.set_raw_header("Referrer-Policy", "strict-origin-when-cross-origin");
    }
}

// Add to rocket builder
.attach(SecurityHeaders)
```

---

## 7. Secure Credential Storage

### Use System Keychain

Add dependency:
```toml
[dependencies]
keyring = "2.0"
```

Implementation:
```rust
use keyring::Entry;

pub fn store_api_key(service: &str, key: &str) -> Result<()> {
    let entry = Entry::new("gemini-cli-desktop", service)?;
    entry.set_password(key)?;
    Ok(())
}

pub fn retrieve_api_key(service: &str) -> Result<String> {
    let entry = Entry::new("gemini-cli-desktop", service)?;
    Ok(entry.get_password()?)
}

pub fn delete_api_key(service: &str) -> Result<()> {
    let entry = Entry::new("gemini-cli-desktop", service)?;
    entry.delete_password()?;
    Ok(())
}
```

---

## 8. Audit Logging

Create comprehensive audit logging:

```rust
use std::fs::OpenOptions;
use std::io::Write;

pub struct AuditLogger {
    log_path: PathBuf,
}

impl AuditLogger {
    pub fn new() -> Result<Self> {
        let log_dir = dirs::home_dir()
            .unwrap()
            .join(".gemini-cli-desktop")
            .join("audit");
        std::fs::create_dir_all(&log_dir)?;
        
        Ok(Self {
            log_path: log_dir.join("security_audit.log"),
        })
    }
    
    pub fn log_event(&self, event_type: &str, details: &str) {
        let timestamp = chrono::Utc::now().to_rfc3339();
        let entry = format!("[{}] {} - {}\n", timestamp, event_type, details);
        
        if let Ok(mut file) = OpenOptions::new()
            .create(true)
            .append(true)
            .open(&self.log_path)
        {
            let _ = file.write_all(entry.as_bytes());
        }
    }
    
    pub fn log_file_access(&self, path: &str, operation: &str) {
        self.log_event("FILE_ACCESS", &format!("{}: {}", operation, path));
    }
    
    pub fn log_command_execution(&self, command: &str) {
        self.log_event("COMMAND_EXEC", command);
    }
    
    pub fn log_auth_attempt(&self, ip: &str, success: bool) {
        self.log_event(
            "AUTH_ATTEMPT",
            &format!("IP: {}, Success: {}", ip, success)
        );
    }
}
```

---

## 9. Configuration File

Create a security configuration file:

```toml
# .gemini-cli-desktop/security.toml

[server]
# Bind address (127.0.0.1 for localhost only, 0.0.0.0 for network)
bind_address = "127.0.0.1"
port = 1858

# Enable authentication
require_auth = true

# Enable TLS (requires cert files)
enable_tls = false
cert_path = ""
key_path = ""

[rate_limiting]
enabled = true
max_requests_per_minute = 60

[file_access]
# Restrict file access to home directory
restrict_to_home = true

# Allowed paths outside home (comma-separated)
allowed_paths = []

[audit]
# Enable security audit logging
enabled = true
log_commands = true
log_file_access = true
log_auth_attempts = true

[commands]
# Enable command validation
validate_commands = true

# Additional blocked command patterns (comma-separated)
blocked_patterns = []
```

---

## 10. Documentation Updates

### Update README.md

Add security section:

```markdown
## Security

### Desktop Application (Recommended)

The desktop application is the most secure way to use Gemini CLI Desktop:
- Runs entirely locally with no network exposure
- Uses Tauri security features
- Requires explicit user permission for all operations

### Web Server Mode

⚠️ **Security Warning**: The web server mode should only be used in trusted environments.

**Security Best Practices:**

1. **Network Isolation**: Only bind to localhost (127.0.0.1)
   ```bash
   GEMINI_BIND_ADDRESS=127.0.0.1 ./gemini-cli-desktop-web
   ```

2. **Firewall Protection**: Ensure port 1858 is blocked from external access
   ```bash
   # Linux (ufw)
   sudo ufw deny 1858
   
   # macOS
   sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add gemini-cli-desktop-web
   ```

3. **API Authentication**: Always use the generated API token
   - Token is stored in `~/.gemini-cli-desktop/api_token`
   - Include in all requests: `Authorization: Bearer <token>`

4. **Regular Updates**: Keep the application updated for security patches

### Reporting Security Issues

Please report security vulnerabilities to: security@piebald.ai

Do not open public issues for security problems.
```

---

## Implementation Priority

### Phase 1: Critical (Immediate)
1. ✅ Change default bind address to 127.0.0.1
2. ✅ Implement API authentication
3. ✅ Add security warnings to documentation

### Phase 2: High Priority (1-2 weeks)
4. ✅ Add path traversal protection
5. ✅ Implement command validation
6. ✅ Add rate limiting
7. ✅ Security headers

### Phase 3: Medium Priority (1 month)
8. ✅ Audit logging
9. ✅ TLS support
10. ✅ System keychain integration

### Phase 4: Ongoing
11. ✅ Regular security audits
12. ✅ Dependency updates
13. ✅ Penetration testing
14. ✅ User security education

---

## Testing Security Fixes

### Test Cases

1. **Network Binding Test**
   ```bash
   # Should only be accessible from localhost
   curl http://localhost:1858/api/check-cli-installed
   # Should work
   
   curl http://192.168.1.x:1858/api/check-cli-installed
   # Should fail (connection refused)
   ```

2. **Authentication Test**
   ```bash
   # Without token - should fail
   curl -X POST http://localhost:1858/api/send-message \
     -H "Content-Type: application/json" \
     -d '{"sessionId":"test"}'
   # Expected: 401 Unauthorized
   
   # With token - should succeed
   curl -X POST http://localhost:1858/api/send-message \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer <token>" \
     -d '{"sessionId":"test"}'
   # Expected: Success
   ```

3. **Path Traversal Test**
   ```bash
   # Should be blocked
   curl -X POST http://localhost:1858/api/read-file-content \
     -H "Authorization: Bearer <token>" \
     -d '{"path":"../../../etc/passwd"}'
   # Expected: Error - path traversal detected
   ```

4. **Rate Limiting Test**
   ```bash
   # Flood requests
   for i in {1..100}; do
     curl http://localhost:1858/api/check-cli-installed &
   done
   # Expected: Some requests return 429 Too Many Requests
   ```

---

**End of Security Recommendations**
