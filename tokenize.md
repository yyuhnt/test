Great! With your detailed tokenizer breakdown, I’ve rewritten the **Tokenization Section** of your **Data Preprocessing chapter** in a more academic and structured way, replacing the previous high-level summary with an in-depth explanation of each header-specific tokenizer.

---

### 2. Domain-Informed Tokenization Strategy

Given the heterogeneous structure of HTTP request and response messages, the preprocessing pipeline applies a **field-specific tokenization framework**. Each tokenizer is tailored to the syntactic and semantic properties of its corresponding HTTP field, enabling the extraction of relevant patterns while abstracting noise and variability. This modular design promotes generalizability, supports attack detection, and enhances the interpretability of learned representations.

#### 2.1 User-Agent Tokenization

User-Agent headers are notoriously diverse, encoding browser identifiers, rendering engines, operating systems, and architecture metadata. The tokenizer addresses this complexity by:

* Splitting on delimiters such as `/` and `:` to isolate versioned components.
* Extracting and processing parenthetical expressions (e.g., operating system, architecture).
* Replacing version numbers with the placeholder `<VER_NUM>`, while preserving canonical browser and OS names.
* Normalizing user-agent variants by reducing lexical entropy (e.g., `Mozilla/5.0`, `AppleWebKit/537.36`).

This approach ensures that tokenized outputs retain semantic structure (browser and platform) without overfitting on exact version strings.

#### 2.2 Accept Header Tokenization

HTTP accept headers describe client preferences for media types, languages, and compression algorithms. The tokenizer handles three subfields:

* **Accept**: Media types and parameters (e.g., `text/html; q=0.9`) are parsed into base types (e.g., `application/json`) and optional quality values, which are abstracted using `<QAL_VAL>`.

* **Accept-Language**: Language and region codes (e.g., `en-US`, `fr-CA`) are tokenized while replacing quality values with `<QAL_VAL>`.

* **Accept-Encoding**: Compression methods (e.g., `gzip`, `br`) are retained in their original form, with quality values again abstracted.

This structured parsing allows the model to distinguish content preferences while discarding lexical variation in preference weights.

#### 2.3 Security Header Tokenization

The `Sec-Fetch-*` headers encode request context in terms of origin, mode, destination, and user activation. Each is treated as a categorical field with well-defined vocabularies:

* `Sec-Fetch-Site`: Values such as `same-origin`, `cross-site`.
* `Sec-Fetch-Mode`: Modes like `cors`, `navigate`.
* `Sec-Fetch-Dest`: Resource types (`document`, `object`).
* `Sec-Fetch-User`: Binary indicator (`?1`, `?0`).

Tokenization maps each value to its canonical form without further abstraction, preserving interpretability for downstream attention mechanisms.

#### 2.4 Cookie and Set-Cookie Tokenization

Cookies carry session data and user context, often obfuscated or base64-encoded. Tokenization is split by function:

* **Cookie** headers extract key–value pairs, recognizing:

  * Session identifiers, abstracted as `<sessionID>`.
  * Encoded values detected via entropy/profile checks and replaced by `<B64_VAL>`.
  * Non-parsable or randomized strings replaced by `<RANDOM>`.

* **Set-Cookie** headers are more structured, including flags and attributes:

  * Domains (`Domain=example.com`) are tokenized with `<domain>`.
  * Expiration times (`Expires=...`) become `<time>`.
  * Path segments and random strings are mapped to `<RAND_STR>`.
  * Security flags (`Secure`, `HttpOnly`) are preserved as-is.

This layered tokenization supports robust session-related anomaly detection and enhances generalization across deployments.

#### 2.5 URL Tokenization

The URL tokenizer decomposes input into structured components:

* Scheme, netloc, and port.
* Path segments (e.g., `/admin/login`) are abstracted using `<RAND_DIR>`.
* Query keys and values are separated and normalized, with volatile names/values mapped to `<RAND_VAR>`.
* Hostnames or IPs are replaced with `<ip>`.
* Fully randomized or externally injected URLs are reduced to `<RAND_URL>`.

Additionally, the tokenizer integrates lightweight heuristics to flag path traversal attempts (`../` sequences) and injection-like substrings.

#### 2.6 Response Body Tokenization

Depending on content-type, different strategies are employed:

* **JSON Bodies**: Parsed hierarchically, mapping:

  * String values to `<RAND_STR>`.
  * Numeric values to `<NUMBER>`.
  * Embedded file paths to `<RAND_DIR>`.

* **HTML Bodies**: HTML tags, attributes, and content are extracted via tree parsing:

  * Links (`href`, `src`) → `<RAND_URL>`.
  * IDs and classes → `<RAND_ID>`.
  * Text content → `<RAND_STR>`.
  * Dimensions or numeric attributes → `<NUMBER>`.

This dual tokenizer captures both semantic content and structural layout for web responses.

#### 2.7 Security-Aware Tokenizers

To enhance detection capability, a security-focused layer scans for known threat signatures:

* **Log4j**: Identifies payloads such as `${jndi:ldap://...}` and replaces the malicious URL with `<RAND_URL>`.

* **RCE/XSS**: Tags patterns such as embedded `<script>` blocks or injected JavaScript with `<RAND_STR>`, highlighting syntactic anomalies.

* **SQL Injection**: Detects SQL grammar (e.g., `SELECT`, `UNION`, `--`) and normalizes values using `<NUMBER>` while preserving key SQL keywords.

These tokenizers operate before general field-level tokenization, ensuring that threats are flagged at the earliest possible stage.

---

Let me know if you'd like a matching **diagram**, **LaTeX version**, or the next section (like vocabulary encoding or feature aggregation).
