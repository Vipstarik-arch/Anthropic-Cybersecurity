# Audit CI / igienizare — Anthropic Cybersecurity Skills

**Dată:** 2026-09-09 · **Ramura:** `arena/01a08575-anthropic-cybersecurity` (bază `54a7988` de pe `main`)
**Obiect:** rularea integrală a celor 5 gate-uri CI pe toate cele 818 skill-uri + cartografierea datoriei de igienizare și un plan de remediere prioritizat.
**Metodă:** executare locală a exact acelorași comenzi din `.github/workflows/validate-skills.yml`, cu venv Python + PyYAML 6.0.3.

> ## ⚠️ Actualizare — Faza 0 executată (2026-09-09)
> La cererea utilizatorului de a trece la implementare efectivă, secțiunile §3.2/§4 de mai jos au fost **aplicate** (nu doar recomandate). Starea curentă după implementare:
> - **Trunchierea descrierilor reparată:** 10 descrieri erau tăiate hard la exact 200 de caractere (în mijlocul cuvântului — bug de truncare real); le-am restaurat **fidel din paragraful `## Overview`** al aceluiași SKILL.md (descrierea existentă era prefix al overview-ului). Alte 3 au primit doar punct final. `desc-ends-punctuation`: **13 → 0**.
> - **Coliziune nouă apărută din restaurare:** restaurarea fidelă a `exploiting-active-directory-with-bloodhound` a scos la suprafață o suprapunere reală cu `performing-active-directory-bloodhound-analysis` (0.494). Am **disambiguat** (nu allowlist): descrierea `exploiting…` reflectă acum corpul real (Phase 3: Exploitation Planning, SharpHound, Cypher, execuție lanț Kerberoasting/ACL/delegare cu OPSEC) și numește explicit ambii frați ca negative-trigger.
> - **Allowlist + negative-triggers:** 6 perechi într-adevăr distincte adăugate în `tools/collision-allowlist.json` (motive = wording-ul negative-trigger-ului), iar ambele membre ale fiecărei perechi au primit negative-trigger în descriere.
> - **Rezultate finale (toate 5 gate-uri VERZI):** coliziuni nerevizuite **55 → 49** (marjă față de cap-ul 55, restaurat din poziția „la limită"); baseline lint **980 → 948** (negative-trigger 769→756, use-when 175→169, punctuație 13→0); `index.json` regenerat.
>
> Notă de proces: `securing-serverless-functions` folosea scalar YAML single-quoted (conține `: `), deci trigger-ele s-au adăugat respectând stilul original al fiecărui fișier; nu s-a rescris structura altor câmpuri. Restaurarea din Overview a fost posibilă doar pentru cele 10 fișiere unde overview-ul restaurează fidel prefixul — celelalte 3 descrieri erau fraze complete, nu trunchieri.

---

## 1. Verdict executiv

Toate cele **5 gate-uri CI trec în prezent (verde)** pe 818 skill-uri, cu **0 eșecuri noi**. Bibliotecile sunt sănătoase și construibile. Însă datele arată **trei riscuri operaționale** care nu blochează CI azi, dar îl fac fragil la următoarea contribuție:

| # | Gate (CI) | Rezultat | 818 skills |
|---|---|---|---|
| 1 | `validate-skill.py --all` (frontmatter) | ✅ PASS | 818 passed / 0 failed |
| 2 | `validate-agentskills.py --strict` | ✅ PASS | 818/818 conforme |
| 3 | `generate-index.py --check` | ✅ PASS | index.json actual (818) |
| 4 | `lint-descriptions.py --all --stats` | ✅ PASS | 0 eșecuri noi (980 grandfathered) |
| 5 | `detect-collisions.py --max-unreviewed 55` | ✅ PASS | 55 nerevizuite = **la limită (cap 55)** |

**Riscuri principale:**
1. **Coliziuni la plafon.** Gate-ul 5 e fix la cap: 55 perechi nerevizuite cu pragul exact 55. Orice skill nou care se suprapune cu unul existent ar trece peste cap → **CI roșu la următoarea contribuție** până se revizuiește o pereche. 94 din 818 skill-uri (11%) sunt implicate într-o coliziune nerevizuită.
2. **Baseline de lint greu de plătit.** 980 încălcări „grandfathered" (concentrate pe **769 lipsă negative-trigger**, **175 lipsă use-when**, 23 corp peste 500 linii, 13 descrieri fără punctuație finală). Nu blochează CI, dar sunt exact factorii care alimentează coliziunile și misrouting-ul.
3. **Cluster-uri reale de duplicat.** Analiza descrierilor relevă grupuri de skill-uri aproape identice (memorie, scheduled tasks, DNS-tunneling, GoPhish, JWT, Certificate Transparency, MITRE Navigator, sigstore) ce cer decizii de consolidare/disambiguare, nu doar allowlist.

---

## 2. Rezultate detaliate per gate

### Gate 1 — `validate-skill.py --all` — ✅ 818/818
Verifică câmpurile obligatorii, `name==folder` (kebab-case), lungimea descrierii, subdomain acceptat, număr de tags, duplicate de nume. Totul verde.

### Gate 2 — `validate-agentskills.py --strict` — ✅ 818/818
Conformitate agentskills.io: `name==directory`, descriere 1..1024 caractere, interzicere cuvinte rezervate, injecție unghiulară. Chei top-level non-standard (acceptate, dar de reținut):
`domain` 818 · `subdomain` 818 · `tags` 818 · `version` 818 · `author` 818 · `mitre_attack` 806 · `nist_csf` 805 · `d3fend_techniques` 139 · `atlas_techniques` 93 · `nist_ai_rmf` 97 · `mitre_f3` 94 · `based_on` 1 · `swc_registry` 1.

### Gate 3 — `generate-index.py --check` — ✅ actual
`index.json` e sincron (818 skill-uri). Fișierul e generat; nu se editează manual.

### Gate 4 — `lint-descriptions.py --all --stats` — ✅ 0 noi / 980 grandfathered
| Regulă | Încălcări | Baselinat | Observație |
|---|---|---|---|
| `name-matches-folder` | 0 | 0 | perfect |
| `desc-max-length` | 0 | 0 | perfect (≤1024) |
| `desc-ends-punctuation` | 13 | 13 | fix trivial, lipsește doar punct final |
| `desc-has-use-when` | 175 | 175 | descrierile n-au clauză „Use when …" |
| `desc-has-negative-trigger` | 769 | 769 | descrierile nu spun pentru ce NU sunt |
| `body-max-lines` | 23 | 23 | corp peste 500 linii |

> 175 + 769 + 23 + 13 = **980** = `_total_grandfathered`. Baseline-ul **nu poate decât să scadă** — orice eșec nou e eroare hard.

### Gate 5 — `detect-collisions.py --max-unreviewed 55` — ✅ la limită
Cosine ≥ 0.45 peste (descriere + slug), cu disambiguarea (negative-trigger) scoasă din scorare.
- **Total perechi:** 56 · **revizuite-distinct:** 1 · **nerevizuite:** 55 · cap CI = 55
- **Skill-uri implicate:** 94 din 818 (11,5%)
- Distribuție scoruri: `0.45–0.49`: 25 · `0.50–0.59`: 29 · `0.60–0.69`: 1 → o singură pereche peste 0.60, majoritatea la 0.45–0.55.
- Unica pereche deja revizuită: `hardening-linux-…` vs `hardening-windows-…` (exemplu bun de allowlist cu motivație care dublează negative-trigger-ul).

---

## 3. Datoria de coliziuni — cartografiere pe cluster-uri

Am citit descrierea completă (parsată cu PyYAML) a tuturor skill-urilor implicate și am grupat perechile. Fiecare cluster are o **dispoziție recomandată**: *consolidare*, *disambiguare* (editare descriere, NU allowlist), sau *allowlist* (perechi într-adevăr distincte).

### 3.1 Clustere cu suprapunere reală → necesită consolidare/disambiguare (prioritate mare)
Aici allowlist-ul ar **ascunde** ambiguitatea, deci e interzis de regulile repo-ului.

**Memory forensics (4 skill-uri, ~3 perechi ≥0.47):**
`analyzing-memory-dumps-with-volatility`, `conducting-memory-forensics-with-volatility`, `performing-memory-forensics-with-volatility3`, `performing-memory-forensics-with-volatility3-plugins`
→ Canonicalul descris complet e Volatility 3. `analyzing-memory-dumps…` și `conducting-memory-forensics-with-volatility` îl dublează cu Volatility (v2 vs v3). **Recomandat:** un singur skill „volatility 3" + referință pentru v2, restul consolidate sau redirectate prin negative-trigger.

**Scheduled tasks (3 skill-uri, ~3 perechi):**
`detecting-malicious-scheduled-tasks-with-sysmon` (0.58/0.47), `hunting-for-suspicious-scheduled-tasks` (0.59/0.58), `hunting-for-scheduled-task-persistence` (0.59/0.47)
→ Toate trei sunt hunts de scheduled tasks pe Event 4698/sysmon, T1053(.005). Diferența reală (detect-alert vs proactive hunt vs persistence-focus) trebuie sculptată în descrieri sau consolidate.

**DNS tunneling / exfiltration (≥6 skill-uri, ~6 perechi):**
`analyzing-dns-logs-for-exfiltration`, `detecting-exfiltration-over-dns-with-zeek`, `hunting-for-dns-tunneling-with-zeek`, `detecting-dns-exfiltration-with-dns-query-analysis`, `performing-dns-tunneling-detection`, `analyzing-certificate-transparency…`
→ Cei mai mari consumatori de perechi. **Recomandat:** separare clară `analyzing-…` (SIEM/DNS logs) vs `…-with-zeek` (Zeek) vs detection-rule engineering; multe pot fi consolidate.

**GoPhish / phishing (3 skill-uri):**
`performing-phishing-simulation-with-gophish` (phishing-defense), `performing-red-team-phishing-with-gophish` (red-teaming, automatizare via Python API), `executing-phishing-simulation-campaign` (pentest generic)
→ Se suprapun pe „rulezi o campanie GoPhish". Dacă rămân separate, descrierile trebuie să se trimită unele pe altele.

**JWT (2 skill-uri, 0.47):** `testing-for-json-web-token-vulnerabilities` vs `testing-jwt-token-security` → duplicat aproape cert; `testing-for…` e cel complet (jwt_tool + Burp JWT Editor, algoritmi, none, kid/jku). **Consolidare.**

**Certificate Transparency (3 skill-uri, ~3 perechi):** `analyzing-certificate-transparency-for-phishing`, `analyzing-tls-certificate-transparency-logs`, `auditing-tls-certificate-transparency-logs` → suprapunere mare; `analyzing-cf-for-phishing` e subset aproape al celorlalte.

**MITRE Navigator / APT (3 skill-uri, ~3 perechi):** `analyzing-apt-group-with-mitre-navigator`, `analyzing-threat-actor-ttps-with-mitre-navigator`, `analyzing-threat-actor-ttps-with-mitre-attack` → primele două construiesc amândouă heatmap Navigator pentru TTP-uri de APT.

**Sigstore / cosign (3 skill-uri):** `implementing-image-provenance-verification-with-cosign`, `implementing-sigstore-for-software-signing`, `verifying-build-provenance-with-slsa-sigstore` → o echipă mică de skill-uri sigstore ce trebuie scoase din suprapunere (sign vs verify vs SLSA).

**Volatility + alte reflectări detect/hunt** (detecting-vs-hunting pe același artefact, ex. NTLM relay, DCSync, process injection, credential dumping, golden ticket, lateral movement) → fiecare „detect vs hunt" pe același Eveniment ID e un duplicat conceptual. Diferența trebuie exprimată în descrieri sau unificată.

### 3.2 Perechi cu diferență reală de scop → candidați legitimi de allowlist
Conform regulii din `tools/detect-collisions.py` și `tools/collision-allowlist.json`, o pereche e allowlist doar când e **într-adevăr distinctă** (scop, platformă sau offensiv vs defensiv). Motivația din allowlist **trebuie** să devină și negative-trigger-ul fiecărei descrieri. Propuneri sigure, susținute de descrierile citite:

| Pereche | Scor | Motivație (dublează negative-trigger) |
|---|---|---|
| `performing-serverless-function-security-review` / `securing-serverless-functions` | 0.56 | Recenzie de securitate (identificare risc) vs hardening/deploy (remediere). Offensiv-asess vs defensiv. |
| `implementing-opa-gatekeeper-for-policy-enforcement` / `implementing-policy-as-code-with-open-policy-agent` | 0.49 | Gatekeeper = admission control pe Kubernetes (Rego/Constraints) vs OPA general în CI/devsecops (non-K8s inclusiv). |
| `implementing-stix-taxii-feed-integration` / `implementing-taxii-server-with-opentaxii` | 0.49 | Consumă/citește un feed STIX/TAXII vs găzduiește (servește) un server TAXII. Client vs server. |
| `analyzing-cobalt-strike-beacon-configuration` / `analyzing-cobaltstrike-malleable-c2-profiles` | 0.46 | Extrage config beacon din PE/memorie vs parsează profile Malleable C2 (AST) pentru semnături de rețea. |
| `analyzing-uefi-bootkit-persistence` / `auditing-uefi-firmware-with-chipsec` | 0.50 | Analiza persistării bootkit-ului (implante SPI/ESP, BlackLotus…) vs audit complet al configurației firmware cu chipsec. |
| `configuring-suricata-for-network-monitoring` / `implementing-network-intrusion-prevention-with-suricata` | 0.46 | IDS/monitorizare pasivă vs IPS inline (NFQueue) care blochează trafic. |
| `implementing-fuzz-testing-in-cicd-with-aflplusplus` / `performing-fuzzing-with-aflplusplus` | 0.46 | Integrare fuzzing în pipeline CI vs fuzzing manual/general. |
| `analyzing-supply-chain-malware-artifacts` / `hunting-for-supply-chain-compromise` | 0.46 | Analiză de artefacte malware (malware-analysis) vs threat-hunt proactiv pe SIEM/EDR. |
| `analyzing-security-logs-with-splunk` / `building-detection-rule-with-splunk-spl` | 0.46 | Investigare incident / corelare în Splunk ES vs engineering de reguli SPL. |
| `detecting-modbus-command-injection-attacks` / `monitoring-scada-modbus-traffic-anomalies` | 0.48 | Detecție comandă-injection specifică vs monitorizare generală anomalii Modbus. (verificare suplimentară recomandată) |

> ⚠️ Nu am **aplicat** allowlist-ul: fiecare intrare e o decizie de conținut care aparține întreținătorului. Lista de mai sus e exact ce s-ar adăuga în `tools/collision-allowlist.json` cu `reason`, și coboară „unreviewed" de la 55 → ~45, eliberând cap-ul CI.

### 3.3 Pergrupe care au nevoie doar de o verificare/decizie umană
`detecting-credential-dumping-techniques` / `detecting-t1003-credential-dumping-with-edr` (0.60); `parsing-artifacts-with-eric-zimmerman-tools` / `performing-windows-artifact-analysis-with-eric-zimmerman-tools` (0.57); `conducting-wireless-network-penetration-test` / `performing-wireless-network-penetration-test` (0.50); `implementing-beyondcorp-zero-trust-access-model` / `implementing-zero-trust-with-beyondcorp` (0.50); `configuring-zscaler-private-access-for-ztna` / `implementing-zero-trust-network-access-with-zscaler` (0.53); `implementing-ebpf-security-monitoring` / `implementing-runtime-security-with-tetragon` (0.49); `performing-network-forensics-with-wireshark` / `performing-network-packet-capture-analysis` (0.49); `hunting-for-living-off-the-land-binaries` / `hunting-for-lolbins-execution-in-endpoint-logs` (0.56); `implementing-threat-intelligence-lifecycle-management` / `managing-intelligence-lifecycle` (0.53); `exploiting-kerberoasting-with-impacket` / `performing-kerberoasting-attack` (0.51); `exploiting-oauth-misconfiguration` / `testing-oauth2-implementation-flaws` (0.46); `exploiting-websocket-vulnerabilities` / `testing-websocket-api-security` (0.53); `detecting-lateral-movement-with-splunk` / `performing-lateral-movement-detection` (0.48); `hunting-for-beaconing-with-frequency-analysis` / `hunting-for-command-and-control-beaconing` (0.45); `performing-purple-team-atomic-testing` / `performing-threat-emulation-with-atomic-red-team` (0.50).

---

## 4. Datoria de lint — plan de plată

Ordinea optimă (cel mai mic risc / cel mai mare efect pe rutare):

1. **`desc-ends-punctuation` (13 skill-uri)** — fix trivial: adaugă punct final la descriere. Atenție: descrierile sunt scalare YAML multiline → editare cu PyYAML, nu cu regex. Redă +1 sănătate (truncation canary).
2. **`desc-has-use-when` (175 skill-uri)** — fiecare descriere trebuie să spună când se declanșează. E deja exact modelul folosit de skill-urile conforme („Use when …, after …, during …"). Fără asta, un agent nu poate alege corect → alimentează și coliziunile.
3. **`desc-has-negative-trigger` (769 skill-uri)** — cel mai mare efort, cel mai mare câștig pe *skill collision*. Regula repo-ului: negative-trigger-ul numește skill-ul frate („Do not use for X — use <other-skill>."). Plata se face **natural** odată cu disambiguarea clusterelor din §3.1 și cu allowlist-urile din §3.2.
4. **`body-max-lines` (23 skill-uri)** — corp > 500 linii. Mutați detaliul în `references/` (exact recomandarea AGENTS.md: „depth belongs in references/").

Fiecare fix se însoțește de `python tools/lint-descriptions.py --update-baseline` (baseline-ul poate doar să scadă) și de regenerarea `index.json` dacă s-a atins vreo descriere.

---

## 5. Observații pe tooling / flux

- **`index.json` e generat** — niciodată editat manual; `update-index.yml` îl regenerează doar pe push la `main` cu căi `skills/**`. Pe PR, gate-ul 3 verifică actualitatea.
- **Autori/licență:** 818/818 `license: Apache-2.0`; concentrare puternică de autor (770 `mahipal`, 30 `mukul975`) → pentru lucrări de igienizare pe scară largă, confirmarea întreținătorului e practic obligatorie înainte de editare în masă.
- **Subdomeniile** folosesc aliasuri tolerante (`red-teaming`/`red-team`, `ot-ics-security`/`ot-security`, `identity-access-management`/`identity-and-access-management` etc.) — sursă de nuanță în clustering, nu o eroare.
- Mediul local: PyYAML lipsea; validarea necesită `python3 -m venv .venv-validate && pip install pyyaml` (PEP 668 blochează install global). Nu am inclus venv-ul în git.

---

## 6. Roadmap prioritar (recomandat)

**Faza 0 — stabilizare (zile):**
- Revizuiește cele 10 perechi din §3.2 → allowlist cu `reason` → coboară `--max-unreviewed` la ~45. Eliberează cap-ul CI imediat.
- Fix cele 13 `desc-ends-punctuation` + `--update-baseline`.

**Faza 1 — de-risc pe cluster (săptămâni):**
- Consolidează cluster-urile §3.1 (memorie, scheduled tasks, DNS, GoPhish, JWT, CT, MITRE-Navigator, sigstore) — fiecare consolidare șterge simultan mai multe perechi și plătește negative-triggers.
- Adaugă `use-when` pe subsetul de skill-uri implicate în coliziuni (cel mai mare efect rutare per editare).

**Faza 2 — plata baseline-ului de lint (continuu):**
- 769 negative-triggers: se plătesc „by-product" al fazei 1; rămânătorii pot fi baleiați descriere cu descriere.
- 175 `use-when` și 23 `body-max-lines`: restante independente.

**Impact așteptat:** unreviewed 55 → ~0 (allowlist corect) sau ~10 (dacă se preferă disambiguare), `desc-ends-punctuation` 13 → 0, și CI rămâne verde cu marjă pentru contribuții noi.

---

## 7. Anexă — comenzi de verificare

```bash
python3 -m venv .venv-validate && . .venv-validate/bin/activate && pip install pyyaml
python tools/validate-skill.py --all
python tools/validate-agentskills.py --strict
python tools/generate-index.py --check
python tools/lint-descriptions.py --all --stats
python tools/detect-collisions.py --max-unreviewed 55
```

Toate au rulat verde la data raportului pe commit-ul de bază.
