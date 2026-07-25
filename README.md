# 🛡️ mcp-security-checklist

> A practical, copy-pasteable hardening checklist for building, installing, and operating **Model Context Protocol (MCP)** servers and client configs — bilingual **English + Türkçe**.
> Her madde bir onay kutusu `[ ]`, bir **neden**, ve **OWASP LLM Top 10 (2025) / MITRE ATLAS** eşlemesi içerir.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![OWASP LLM Top 10](https://img.shields.io/badge/OWASP-LLM%20Top%2010%20(2025)-000000.svg)](https://genai.owasp.org/llm-top-10/)
[![MITRE ATLAS](https://img.shields.io/badge/MITRE-ATLAS-red.svg)](https://atlas.mitre.org/)
[![Languages: EN | TR](https://img.shields.io/badge/lang-EN%20%7C%20TR-informational.svg)](#)
[![MCP Hardening](https://img.shields.io/badge/Model%20Context%20Protocol-hardening-6E56CF.svg)](https://modelcontextprotocol.io/)
[![Scan with uncloak](https://img.shields.io/badge/scan%20with-uncloak-8A2BE2.svg)](https://github.com/fevziegeyurtsevenler/uncloak)

**This is a community checklist, not a certification or a guarantee.** Passing every item reduces risk; it does not make a deployment "secure." Treat MCP tool descriptions, tool outputs, and server code as **untrusted input** by default.
**Bu bir sertifika ya da garanti değildir.** Tüm maddeleri geçmek riski azaltır; bir kurulumu "güvenli" yapmaz. MCP araç tanımlarını, araç çıktılarını ve sunucu kodunu **varsayılan olarak güvenilmez girdi** kabul edin.

---

## Why MCP needs its own checklist / Neden MCP'ye özel bir liste?

MCP connects LLM agents to tools, files, and external systems. That connection is exactly where the LLM-specific supply-chain and prompt-injection risks land. A tool description the model reads is an **instruction channel**; a server you install is **code running with your agent's authority**. Several of these paths already have real CVEs:

MCP, LLM ajanlarını araçlara, dosyalara ve dış sistemlere bağlar — ve LLM'e özgü tedarik zinciri ile prompt injection riskleri tam da burada yoğunlaşır. Aşağıdakiler **doğrulanmış, gerçek** zafiyetlerdir:

| CVE | Component | Class | CVSS | Note |
|---|---|---|---|---|
| [CVE-2025-49596](https://nvd.nist.gov/vuln/detail/CVE-2025-49596) | MCP Inspector `< 0.14.1` | Missing auth → RCE (CWE-306) | **9.4** | Unauthenticated requests could launch MCP commands over stdio (DNS-rebinding assisted). |
| [CVE-2025-6514](https://nvd.nist.gov/vuln/detail/CVE-2025-6514) | `mcp-remote` `0.0.5`–`0.1.15` | OS command injection (CWE-78) | **9.6** | Triggered when connecting to an **untrusted** MCP server via a crafted `authorization_endpoint` URL. |
| [CVE-2025-53110](https://nvd.nist.gov/vuln/detail/CVE-2025-53110) | MCP **Filesystem** server (`< 0.6.4` / `2025.7.01`) | Path traversal (CWE-22) | **7.3** | An allowed-directory **prefix match** let access escape the restricted path. |

> These are examples of *classes* of MCP risk (unauthenticated control planes, exec/command injection, over-broad filesystem scope), not an exhaustive list. Check the current advisory for any server you run.

---

## Quick start: scan before you trust / Güvenmeden önce tara

The very first hardening step is a **pre-install scan** of the server source and config with [**uncloak**](https://github.com/fevziegeyurtsevenler/uncloak) — it surfaces invisible-Unicode instruction smuggling and prompt-injection / supply-chain risks hidden in Skills, MCP servers, and rules files.

```bash
# 1) Fetch the server WITHOUT running its install scripts
npm install --ignore-scripts   # or: pip download --no-deps / git clone (no build)

# 2) Scan the source tree and any config (mcp.json / claude_desktop_config.json)
#    See uncloak's README for the exact subcommands/flags in your version.
npx uncloak scan ./path/to/mcp-server
npx uncloak scan ./mcp.json

# 3) Only then install/enable — in a sandbox first
```

If uncloak flags hidden instructions in a tool description or config, **do not install** until you understand why they are there. Bir araç tanımında ya da konfigürasyonda gizli talimat bulunursa, nedenini anlamadan **kurmayın**.

---

## How to use this checklist / Nasıl kullanılır

- Copy the relevant section into your PR description or a `SECURITY.md` and tick items as you verify them.
- Each item is: `- [ ] Action` + **Why / Neden** + **Maps to** (OWASP LLM ID · ATLAS technique).
- Treat unchecked **P0** items as release blockers.
- İlgili bölümü PR açıklamanıza kopyalayın; doğruladıkça kutuları işaretleyin. İşaretlenmemiş **P0** maddeleri sürüm engeli sayın.

**OWASP LLM Top 10 (2025)** IDs used below: `LLM01 Prompt Injection` · `LLM02 Sensitive Information Disclosure` · `LLM03 Supply Chain` · `LLM04 Data & Model Poisoning` · `LLM05 Improper Output Handling` · `LLM06 Excessive Agency` · `LLM07 System Prompt Leakage` · `LLM08 Vector & Embedding Weaknesses` · `LLM09 Misinformation` · `LLM10 Unbounded Consumption`.

---

## 1. Pre-install scanning / Kurulum-öncesi tarama

- [ ] **Scan source + config with `uncloak` before installing.** *Kurmadan önce kaynak ve config'i `uncloak` ile tara.*
  - **Why / Neden:** Tool descriptions and rules files can carry invisible-Unicode or injected instructions that execute in the agent's context. *Araç tanımları ve kural dosyaları, ajan bağlamında çalışan görünmez-Unicode/enjekte talimatlar taşıyabilir.*
  - **Maps to:** LLM01, LLM03 · ATLAS AML.T0051.001 (Indirect Prompt Injection), AML.T0010 (ML Supply Chain Compromise)
- [ ] **Pin an exact version or commit; never `@latest`.** *Tam sürüm/commit sabitle; asla `@latest` kullanma.*
  - **Why:** A moving version lets a compromised or updated package change behavior after your review ("rug pull"). *Kayan sürüm, incelemenden sonra davranışın değişmesine izin verir.*
  - **Maps to:** LLM03 · AML.T0010
- [ ] **Block install/postinstall scripts by default (`--ignore-scripts`).** *Kurulum betiklerini varsayılan olarak engelle.*
  - **Why:** `postinstall` runs arbitrary code before you ever review the server. *`postinstall`, sen incelemeden önce keyfi kod çalıştırır.*
  - **Maps to:** LLM03 · AML.T0010
- [ ] **Verify publisher/provenance; stars and downloads are not trust.** *Yayıncı/kaynağı doğrula; yıldız ve indirme sayısı güven değildir.*
  - **Why:** Typosquats and look-alike servers impersonate popular ones. *Typosquat ve benzer-isimli sunucular popülerlerini taklit eder.*
  - **Maps to:** LLM03 · AML.T0010
- [ ] **Check the server's CVE/advisory history** (e.g. `mcp-remote`, MCP Inspector, filesystem server). *Sunucunun CVE geçmişini kontrol et.*
  - **Why:** Known-vulnerable versions are still widely deployed. *Bilinen zafiyetli sürümler hâlâ yaygın kullanımda.*
  - **Maps to:** LLM03
- [ ] **First run in a sandbox/container with no secrets and no network.** *İlk çalıştırmayı sırlar ve ağ olmadan sandbox/konteynerde yap.*
  - **Why:** Contains exec, exfiltration, and path-traversal behavior during evaluation. *Değerlendirme sırasında exec, sızdırma ve dizin-aşımı davranışını sınırlar.*
  - **Maps to:** LLM06, LLM10 · AML.T0053 (LLM Plugin Compromise)

---

## 2. Tool-definition hygiene / Araç-tanımı hijyeni

- [ ] **Read every tool name/description/schema as untrusted attacker-controlled text.** *Her araç adı/açıklaması/şemasını saldırgan-kontrollü metin gibi oku.*
  - **Why:** "Tool poisoning" hides instructions inside descriptions the model obeys but the user never sees. *"Tool poisoning", kullanıcının görmediği ama modelin uyduğu talimatları açıklamalara gizler.*
  - **Maps to:** LLM01 · AML.T0051 (LLM Prompt Injection), AML.T0053
- [ ] **Reject hidden/zero-width/bidi Unicode in tool metadata (scan with `uncloak`).** *Araç metadatasındaki gizli/sıfır-genişlik/bidi Unicode'u reddet.*
  - **Why:** Invisible characters smuggle instructions past human review. *Görünmez karakterler, talimatları insan incelemesinden kaçırır.*
  - **Maps to:** LLM01, LLM03 · AML.T0051.001
- [ ] **Detect "line jumping": text that tries to act before the tool is invoked.** *Araç çağrılmadan etkili olmaya çalışan metni tespit et.*
  - **Why:** Descriptions loaded into context can steer the agent without any tool call. *Bağlama yüklenen açıklamalar, hiçbir araç çağrısı olmadan ajanı yönlendirebilir.*
  - **Maps to:** LLM01 · AML.T0051
- [ ] **Pin and hash tool definitions; re-approve on any change (anti "rug pull").** *Araç tanımlarını sabitle ve hash'le; her değişimde yeniden onayla.*
  - **Why:** A server can serve benign tools at approval time and swap them later. *Sunucu, onay anında zararsız araç sunup sonra değiştirebilir.*
  - **Maps to:** LLM03, LLM06 · AML.T0010, AML.T0053
- [ ] **Namespace tools per server; prevent cross-server tool shadowing.** *Araçları sunucu bazında isim-uzayına al; sunucular-arası gölgelemeyi engelle.*
  - **Why:** A malicious server can redefine or intercept a trusted server's tool. *Zararlı sunucu, güvenilir aracı yeniden tanımlayabilir/araya girebilir.*
  - **Maps to:** LLM03 · AML.T0053
- [ ] **Treat tool *output* that re-enters the model as untrusted (secondary injection).** *Modele geri dönen araç çıktısını da güvenilmez say.*
  - **Why:** Fetched web/file content is a classic indirect-injection vector. *Çekilen web/dosya içeriği klasik dolaylı-enjeksiyon vektörüdür.*
  - **Maps to:** LLM01, LLM05 · AML.T0051.001

---

## 3. Command / exec hardening / Komut ve exec sertleştirme

- [ ] **Never pass model output straight into a shell.** *Model çıktısını doğrudan shell'e verme.*
  - **Why:** This is the root cause pattern behind command-injection CVEs like CVE-2025-6514. *CVE-2025-6514 gibi komut-enjeksiyonlarının kök nedeni budur.*
  - **Maps to:** LLM05, LLM06 · AML.T0053
- [ ] **Use argument arrays, not string-built commands; avoid `shell=True`.** *Dize ile kurulan komut yerine argüman dizisi kullan; `shell=True`'dan kaçın.*
  - **Why:** Prevents metacharacter (`; | & $()`) injection. *Metakarakter enjeksiyonunu önler.*
  - **Maps to:** LLM05 · AML.T0053
- [ ] **Allowlist executable binaries; deny arbitrary "run this command" tools.** *Çalıştırılabilir ikili dosyaları allowlist'le; keyfi "şu komutu çalıştır" araçlarını reddet.*
  - **Why:** An open exec surface turns any injection into RCE. *Açık exec yüzeyi, her enjeksiyonu RCE'ye çevirir.*
  - **Maps to:** LLM06 · AML.T0053
- [ ] **Validate/escape all inputs; reject shell metacharacters and control bytes.** *Tüm girdileri doğrula/escape'le.*
  - **Why:** Defense in depth even when using arg arrays. *Argüman dizisi kullansan bile katmanlı savunma.*
  - **Maps to:** LLM05
- [ ] **Enforce timeouts, output-size caps, and CPU/memory limits on every exec.** *Her exec'e zaman aşımı, çıktı-boyutu ve CPU/bellek sınırı koy.*
  - **Why:** Blocks resource-exhaustion / unbounded-consumption abuse. *Kaynak-tüketimi istismarını engeller.*
  - **Maps to:** LLM10

---

## 4. Permission / scope / Yetki ve kapsam

- [ ] **Least privilege: mount only the exact directories the server needs.** *En az yetki: sunucuya yalnızca ihtiyaç duyduğu dizinleri ver.*
  - **Why:** Over-broad filesystem scope caused CVE-2025-53110 (prefix-match path traversal). *Aşırı geniş dosya kapsamı CVE-2025-53110'a yol açtı.*
  - **Maps to:** LLM02, LLM06 · AML.T0057 (LLM Data Leakage)
- [ ] **Deny-by-default tool allowlist per agent/workspace.** *Ajan/çalışma-alanı başına varsayılan-reddet allowlist.*
  - **Why:** New/unknown tools should never auto-enable. *Yeni/bilinmeyen araçlar asla otomatik etkinleşmemeli.*
  - **Maps to:** LLM06 · AML.T0053
- [ ] **Require human-in-the-loop confirmation for destructive or write actions.** *Yıkıcı/yazma işlemleri için insan onayı iste.*
  - **Why:** Excessive agency without a gate turns a bad suggestion into a bad action. *Kapısız aşırı-yetki, kötü öneriyi kötü eyleme çevirir.*
  - **Maps to:** LLM06 · AML.T0054 (LLM Jailbreak)
- [ ] **Separate read vs write capabilities into distinct, individually-granted scopes.** *Okuma ve yazma yeteneklerini ayrı, tek tek verilen kapsamlara ayır.*
  - **Why:** Read-only tasks should never carry write authority. *Salt-okunur görev yazma yetkisi taşımamalı.*
  - **Maps to:** LLM06
- [ ] **No ambient authority — bind capabilities to the requesting user/session.** *Ortam-yetkisi yok — yetenekleri isteyen kullanıcı/oturuma bağla.*
  - **Why:** Prevents one user's session from using another's privileges. *Bir oturumun başkasının ayrıcalıklarını kullanmasını önler.*
  - **Maps to:** LLM06, LLM02

---

## 5. Token / OAuth / Kimlik ve OAuth

- [ ] **Never hardcode secrets in `mcp.json` / config; use env or a secret manager.** *Sırları config'e gömme; env veya secret manager kullan.*
  - **Why:** Configs get committed, synced, and shared. *Config'ler commit'lenir, senkronlanır, paylaşılır.*
  - **Maps to:** LLM02
- [ ] **Follow the MCP authorization spec: OAuth 2.1, PKCE, exact redirect URIs.** *MCP yetkilendirme spec'ini uygula: OAuth 2.1, PKCE, tam redirect URI.*
  - **Why:** Loose redirect matching enables token theft. *Gevşek redirect eşleşmesi token hırsızlığına yol açar.*
  - **Maps to:** LLM02, LLM06 · AML.T0053
- [ ] **Use Resource Indicators (RFC 8707) and validate token `audience`.** *Resource Indicators (RFC 8707) kullan ve token `audience`'ını doğrula.*
  - **Why:** Stops "confused deputy" / token pass-through to the wrong resource. *"Confused deputy"/yanlış kaynağa token aktarımını durdurur.*
  - **Maps to:** LLM06 · AML.T0053
- [ ] **Issue short-lived, narrowly-scoped tokens; rotate and revoke.** *Kısa ömürlü, dar kapsamlı token ver; döndür ve iptal et.*
  - **Why:** Limits blast radius of a leaked credential. *Sızan kimlik bilgisinin etki alanını sınırlar.*
  - **Maps to:** LLM02
- [ ] **Redact tokens from logs, errors, and telemetry.** *Token'ları log, hata ve telemetriden gizle.*
  - **Why:** Secrets leak most often through observability, not attackers. *Sırlar en sık gözlemlenebilirlik yoluyla sızar.*
  - **Maps to:** LLM02, LLM07

---

## 6. Network / egress / Ağ ve dışa-çıkış

- [ ] **Bind local transports to `127.0.0.1`, never `0.0.0.0`.** *Yerel taşımaları `127.0.0.1`'e bağla, `0.0.0.0`'a değil.*
  - **Why:** An exposed, unauthenticated control plane is exactly the CVE-2025-49596 pattern. *Açık, kimliksiz kontrol düzlemi tam da CVE-2025-49596 desenidir.*
  - **Maps to:** LLM06 · AML.T0053
- [ ] **Validate `Origin` and defend against DNS rebinding on HTTP/SSE transports.** *HTTP/SSE taşımalarında `Origin` doğrula ve DNS rebinding'e karşı koru.*
  - **Why:** A browser page can otherwise reach a localhost MCP server. *Aksi halde bir tarayıcı sayfası localhost MCP sunucusuna erişebilir.*
  - **Maps to:** LLM06
- [ ] **Default-deny egress; allowlist only required destinations.** *Varsayılan-reddet egress; yalnızca gerekli hedefleri allowlist'le.*
  - **Why:** Cuts off the data-exfiltration path after an injection. *Enjeksiyon sonrası veri-sızdırma yolunu keser.*
  - **Maps to:** LLM02 · AML.T0025 (Exfiltration via Cyber Means), AML.T0057
- [ ] **Require TLS and verify certificates for all outbound calls.** *Tüm dış çağrılar için TLS zorunlu kıl ve sertifika doğrula.*
  - **Why:** Prevents MITM and downgrade on credentials/data. *Kimlik/veri üzerinde MITM ve downgrade'i önler.*
  - **Maps to:** LLM02
- [ ] **Rate-limit and quota outbound requests.** *Dış istekleri hız-sınırla ve kotaya bağla.*
  - **Why:** Contains exfiltration loops and cost/DoS abuse. *Sızdırma döngülerini ve maliyet/DoS istismarını sınırlar.*
  - **Maps to:** LLM10

---

## 7. Monitoring & response / İzleme ve müdahale

- [ ] **Log every tool invocation with (redacted) arguments and results.** *Her araç çağrısını (gizlenmiş) argüman ve sonuçlarıyla logla.*
  - **Why:** Without an audit trail, injection and misuse are invisible. *Denetim izi olmadan enjeksiyon ve kötüye kullanım görünmez.*
  - **Maps to:** LLM02, LLM07
- [ ] **Alert on tool-definition or capability changes at runtime.** *Çalışma-zamanında araç-tanımı/yetenek değişikliklerinde uyarı ver.*
  - **Why:** This is how you catch a rug pull in production. *Üretimde bir "rug pull"ı böyle yakalarsın.*
  - **Maps to:** LLM03, LLM06 · AML.T0010
- [ ] **Monitor for anomalous egress volume and new destinations.** *Anormal dışa-çıkış hacmi ve yeni hedefleri izle.*
  - **Why:** Data exfiltration shows up as unexpected outbound traffic. *Veri sızıntısı beklenmedik dış trafik olarak görünür.*
  - **Maps to:** LLM02 · AML.T0025
- [ ] **Keep a kill switch to disable any single server instantly.** *Herhangi bir sunucuyu anında kapatacak bir kill switch tut.*
  - **Why:** Fast containment beats slow investigation during an incident. *Olay anında hızlı kontrol, yavaş incelemeyi yener.*
  - **Maps to:** LLM06
- [ ] **Re-scan servers/configs with `uncloak` in CI and on every update.** *CI'da ve her güncellemede sunucu/config'i `uncloak` ile yeniden tara.*
  - **Why:** Updates re-open supply-chain and injection risk; make scanning continuous. *Güncellemeler riski yeniden açar; taramayı sürekli kıl.*
  - **Maps to:** LLM03, LLM01 · AML.T0010, AML.T0051.001

---

## OWASP LLM Top 10 (2025) ↔ ATLAS coverage map

| Checklist section | Primary OWASP LLM (2025) | MITRE ATLAS |
|---|---|---|
| 1. Pre-install scanning | LLM03, LLM01 | AML.T0010, AML.T0051.001 |
| 2. Tool-definition hygiene | LLM01, LLM03, LLM05 | AML.T0051, AML.T0053 |
| 3. Command / exec | LLM05, LLM06, LLM10 | AML.T0053 |
| 4. Permission / scope | LLM06, LLM02 | AML.T0053, AML.T0057, AML.T0054 |
| 5. Token / OAuth | LLM02, LLM06, LLM07 | AML.T0053 |
| 6. Network / egress | LLM02, LLM06, LLM10 | AML.T0025, AML.T0057 |
| 7. Monitoring & response | LLM02, LLM03, LLM07 | AML.T0010, AML.T0025 |

> **Verification note.** Identifiers follow the **OWASP Top 10 for LLM Applications (2025)** and the **MITRE ATLAS** matrix. Standards evolve — confirm each ID against the latest published version before citing it in an audit. *Standartlar değişir; bir denetimde alıntılamadan önce her ID'yi güncel sürüme karşı doğrulayın.*

---

## Related work / İlgili çalışmalar

Part of a set of open resources on **AI agent extension supply-chain security** by [Fevzi Ege Yurtsevenler](https://github.com/fevziegeyurtsevenler):

- **[uncloak](https://github.com/fevziegeyurtsevenler/uncloak)** — Reveals hidden prompt-injection & supply-chain risks in AI agent extensions; scans Skills, MCP servers & rules files for invisible-Unicode instruction smuggling. *(Used in the scan steps above.)*
- **[skills-in-the-wild](https://github.com/fevziegeyurtsevenler/skills-in-the-wild)** — An open, reproducible security audit of 3,168 real public AI agent extensions (skills, MCP configs, rules files): dataset + findings + method.
- **[awesome-agent-supply-chain-security](https://github.com/fevziegeyurtsevenler/awesome-agent-supply-chain-security)** — Curated tools, research, standards, and datasets for securing AI agent extensions.
- **[llm-security-skills](https://github.com/fevziegeyurtsevenler/llm-security-skills)** — Agent Skills that turn your AI coding agent into an LLM security reviewer (prompt-injection testing, OWASP LLM Top 10 audits, MCP & RAG review).
- **[prompt-injection-corpus](https://github.com/fevziegeyurtsevenler/prompt-injection-corpus)** — A multilingual (EN + TR) corpus of prompt-injection & jailbreak techniques, each paired with its defense and mapped to OWASP LLM Top 10 & MITRE ATLAS.

---

## References / Kaynaklar

- OWASP — [Top 10 for LLM Applications (2025)](https://genai.owasp.org/llm-top-10/)
- MITRE — [ATLAS matrix](https://atlas.mitre.org/) (techniques `AML.Txxxx`)
- Model Context Protocol — [specification & authorization](https://modelcontextprotocol.io/)
- NVD — [CVE-2025-49596](https://nvd.nist.gov/vuln/detail/CVE-2025-49596) (MCP Inspector), [CVE-2025-6514](https://nvd.nist.gov/vuln/detail/CVE-2025-6514) (`mcp-remote`), [CVE-2025-53110](https://nvd.nist.gov/vuln/detail/CVE-2025-53110) (Filesystem server)
- RFC 8707 — Resource Indicators for OAuth 2.0

---

## Contributing / Katkı

Issues and PRs are welcome — new checklist items must include a **why** and an **OWASP/ATLAS mapping**, and any CVE or metric must link to a primary source. *Yeni maddeler bir **neden** ve **OWASP/ATLAS eşlemesi** içermeli; her CVE/metrik birincil kaynağa bağlanmalı.*

## Author / Yazar

**Fevzi Ege Yurtsevenler** — multilingual LLM/AI security researcher, [AltaySec](https://altaysec.com.tr) · OWASP GenAI (merged contributor) · GitHub [@fevziegeyurtsevenler](https://github.com/fevziegeyurtsevenler)

## License

[MIT](LICENSE) — use it, fork it, ship it. No warranty; this checklist reduces risk but does not certify security.
