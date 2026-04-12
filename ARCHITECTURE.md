# Architecture Overview: Windows Login Extractor (WinLogEx)

## 1. Project Summary
Windows Login Extractor (WinLogEx) is a specialized digital forensics and incident response (DFIR) tool designed to automate the extraction, normalization, and correlation of Windows authentication events. Unlike manual log review in Windows Event Viewer—which can be overwhelming due to the sheer volume of noise and lack of context—WinLogEx stitches together disparate logon, logoff, and session events into a coherent user timeline. This allows analysts to quickly identify session durations, remote connection origins, and suspicious login patterns that would otherwise remain hidden in thousands of individual XML entries.

## 2. Goals & Scope
- **Goals**:
    - Automate extraction of security event logs (4624, 4625, 4634, etc.).
    - Correlate authentication events across different providers (Security, RDP, Registry).
    - Provide a unified timeline of user activity for deeper forensic insight.
    - Flag anomalies like brute force attacks, impossible travel, or lateral movement patterns.
- **Out of Scope**:
    - Real-time EDR-style blocking or active intrusion prevention.
    - Broad host forensics (e.g., file system carving, browser history) outside of authentication artifacts.
    - Active remediation such as password resets or account locking.
- **Target User**: Forensic analysts, incident responders, and security researchers investigating host compromise or unauthorized access.

## 3. Supported Authentication Mechanisms
- **Local Account Authentication**: Tracking of local user logons using NTLM or the local Security Accounts Manager (SAM).
- **Microsoft Account Authentication**: Handling modern Windows 10/11 online-linked account logons.
- **Remote Desktop Protocol (RDP)**: Specialized parsing for RDP-specific logs (e.g., RemoteConnectionManager) mapped to Security event IDs.
- **Active Directory / Kerberos**: Extraction of domain-based authentication details, including TGT/TGS requests and domain controller interactions.

## 4. High-Level Data Flow
The system follows a linear pipeline to ensure data integrity and modularity:

**Raw Artifacts** → **Parsers** → **Normalizer** → **Correlator** → **Anomaly Detector** → **Reporter** → **Output**

1.  **Collection**: Raw `.evtx` files and Registry hives are identified.
2.  **Parsing**: Low-level formats are converted into Python dictionaries.
3.  **Normalization**: Diverse audit data is mapped to a standardized `LogonEvent` model.
4.  **Correlation**: Events are linked by Logon ID and timestamp to build session blocks.
5.  **Detection**: The rule engine scans for suspicious sequences of events.
6.  **Reporting**: Data is serialized into multiple formats for human or machine consumption.

## 5. Module Breakdown

### 5.1 Parsers (`src/parsers/`)
- `evtx_parser.py`: Extracts XML data from binary Windows Event Log files.
- `registry_parser.py`: Queries registry hives for persistent authentication traces (e.g., Terminal Services settings).
- `rdp_parser.py`: Focuses on RDP-specific event logs, extracting Client Name and IP from connection logs.
- `normalizer.py`: Standardizes diverse schema fields into a unified internal representation.

### 5.2 Correlator (`src/correlator/`)
- `correlator.py`: Orchestrates the linking process between different data sources.
- `session_stitcher.py`: Pairs Logon (4624) and Logoff (4634) events to calculate session duration.
- `timeline_builder.py`: Sorts all normalized events into a chronological, per-user sequence.

### 5.3 Anomaly Detector (`src/detector/`)
- `anomaly_detector.py`: The engine that iterates through the timeline and triggers alerts.
- `rules.py`: A library of detection signatures (e.g., "Multiple Failures followed by Success").

### 5.4 Reporter (`src/reporter/`)
- `reporter.py`: Manages the export pipeline and template rendering.
- `json_exporter.py` / `csv_exporter.py`: Data serialization for integration with other tools.
- `html_exporter.py`: Generates interactive dashboards with visualization.
- `md_exporter.py`: Produces clean summaries optimized for documentation.

### 5.5 Utilities (`src/utils/`)
- `models.py`: Defines the central `LogonEvent` dataclass and other shared schemas.
- `config.py`: Global configuration for file paths, severity thresholds, and timezones.
- `logger.py`: Internal system logging for debugging tool operations.

## 6. Core Data Model
The `LogonEvent` dataclass is the primary object that flows through the pipeline:

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | `datetime` | The UTC time of the event. |
| `event_id` | `int` | The Windows Event ID (e.g., 4624, 4625). |
| `username` | `str` | The target user account name. |
| `domain` | `str` | The account domain or workstation name. |
| `logon_type` | `int` | Logon type code (e.g., 2 for Local, 10 for RDP). |
| `logon_id` | `str` | Unique hex ID assigned to the session. |
| `source_ip` | `str` | Remote IP address for network/RDP logons. |
| `workstation` | `str` | The name of the machine where the logon originated. |
| `auth_package` | `str` | The protocol used (Kerberos, NTLM, etc.). |
| `status` | `str` | Success or a hex error code for failures. |

## 7. Artifact Sources
- **Security Logs**: `C:\Windows\System32\winevt\Logs\Security.evtx`
- **RDP Operational Logs**: `Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational.evtx`
- **Registry Hives**: `SYSTEM`, `SOFTWARE`, and `SAM` (for local account info).
- **Execution Mode**:
    - **Live System**: Reads directly from local paths (requires Admin/System privileges).
    - **Forensic Image**: Reads from a directory of files extracted from a mounted image.

## 8. Correlation Logic
- **Logon ID Linking**: Uses the 64-bit `TargetLogonId` to correlate a Logon (4624) with its specific Logoff (4634/4647), regardless of how many other users are logged in.
- **Cross-Source Linking**: Matches the timestamp and Source IP from RDP-specific logs (1149) with the ensuing Security logon (4624) to provide a complete view of remote access.
- **Session Duration Calculation**: Computed by subtracting the logon timestamp from the linked logoff timestamp.

## 9. Anomaly Detection Rules
| Rule | Trigger Condition | Severity |
|------|------------------|----------|
| **Brute Force** | >5 failed logons (4625) for one user in <60 seconds. | High |
| **Impossible Travel** | Logons from different geographic locations within an impossible timeframe. | Critical |
| **RDP Tunneling** | RDP logon (Type 10) originating from `localhost` (127.0.0.1). | High |
| **Privilege Escalation** | Successful logon (4624) followed by sensitive privilege assignment (4672). | High |

## 10. Output Formats
- **JSON**: Hierarchical structured data for SIEM/SOAR ingestion.
- **CSV**: Spreadsheet-compatible format for manual sorting and filtering.
- **HTML**: Premium visual report featuring session graphs and severity highlights.
- **Markdown**: Summary report suitable for Git commit logs or case notes.

## 11. Technology Stack
| Component | Technology | Reason |
|-----------|-----------|--------|
| **Language** | Python 3.10+ | Standard for DFIR automation with extensive library support. |
| **Data Analysis** | Pandas | High-performance filtering and session correlation. |
| **Parsing** | python-evtx | Robust handling of binary Windows XML event structures. |
| **Visualization** | Plotly / Jinja2 | Dynamic dashboard generation and template rendering. |

## 12. Directory Structure
```text
Windows-Login-Extractor/
├── ARCHITECTURE.md          # This document
├── README.md                # Project resources and setup
├── src/                     # Source code root
│   ├── main.py              # Application entry point
│   ├── parsers/             # Artifact extraction and normalization
│   ├── correlator/          # Session stitching and timeline logic
│   ├── detector/            # Rule-based anomaly engine
│   ├── reporter/            # Export and visualization logic
│   └── utils/               # Shared models, logging, and config
├── tests/                   # Unit, integration, and scenario tests
├── docs/                    # Detailed documentation and event ID references
└── data-samples/            # Sample artifacts for testing
```

## 13. Future Work / Limitations
- **Limitations**:
    - Currently requires administrative privileges for live system access.
    - Large volume logs (>1GB) may require optimized chunked parsing.
- **Future Enhancements**:
    - **Machine Learning**: Integration of Isolation Forests for baseline user behavior profiling.
    - **Memory Integration**: Correlating event logs with live memory (RAM) process trees.
    - **Cloud Support**: Parsing logs from Azure AD and AWS Workspace authentication.
