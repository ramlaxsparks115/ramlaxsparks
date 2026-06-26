Error Mapping Design — Experience API
RDPP Payment Domestic Transfer API
Feature: Generic, configuration-driven downstream error mapping framework
Author: RDPP Team
Version: 1.6
Date: 26 June 2026
Status: Implemented

Table of Contents
Problem Statement
Requirements
Architecture Overview
Execution Point in Codebase
Error Code Namespaces
Class Design
Configuration — error-mappings.yaml
Configuration Storage Strategy
Mapping Algorithm
Sample Mapped Output
Integration into CustomResponseBodyAdvice
New TransferErrorCodes Entries Required
File Inventory
Best Practices & Pitfalls
Implementation Checklist
1. Problem Statement
The Experience API orchestrates calls to multiple downstream systems (CPH, Deposits, TransferAdapter, EBBS).
Each downstream system returns its own proprietary error structure and error codes:

CPH Error Response:


{
  "errors": [
    {
      "id": "646abc68-d576-40ed-9923-4eb41b8ad476",
      "code": "VALIDATION-003",
      "detail": "Business Rule failed: The immediate transaction transfer date must be today."
    }
  ]
}
Deposits Error Response (thrown by payment-lib-deposits library — DepositsErrorCode enum):


{
  "errors": [
    {
      "id": "9e96ac54-a1aa-4fc7-b10b-4cf2cdcef130",
      "code": "DP-LIB-API-202",
      "detail": "No account information found for the given criteria."
    }
  ]
}
Note: The Deposits downstream errors are wrapped by the payment-lib-deposits library
(com.sc.rdpp.deposits.exception.DepositsErrorCode) before reaching the Experience API.
The Experience API therefore receives DP-LIB-API-XXX codes, not raw CASA API codes.

The library places the raw downstream response body in args as
Map.of("downstream", responseBody) rather than exposing it directly at args[0].
This prevents extractDownstreamCode from mining the raw downstream code
(e.g. CUS0001) and ensures the YAML lookup uses the lib's own DP-LIB-API-XXX code.

Issues with the current approach
Issue	Detail
Raw codes exposed	Downstream codes (VALIDATION-003, DP-LIB-API-202) reach the client
No user-friendly messages	Technical backend detail surfaced to end users
Hardcoded per-system handling	Each system requires code changes to handle new errors
No fallback	Unknown/unmapped codes have no safe default
EBBS is isolated	EbbsErrorCodeMapper exists but is not a generalised framework
2. Requirements
#	Requirement
R1	Map each downstream error code to a DT-X-API-XXX or DT-X-EBBS-XXX Experience API error code
R2	Support mapping per system (CPH, DEPOSITS, EBBS, FXRATES, etc.)
R3	Support exact code match → system-level default → global fallback
R4	Configuration-driven — no code changes to add new codes or systems
R5	User-facing description must be client-friendly; raw backend detail preserved in args
R6	Each mapping entry carries the HTTP status code to return
R7	Log and metric every unmapped error code for observability
R8	Align with existing TransferErrorCodes (DT-X-API-XXX) and EbbsTransferErrorCodes (DT-X-EBBS-XXX) enum conventions
3. Architecture Overview

Client
  │
  ▼
Experience API
  ├── Calls Downstream System (CPH / Deposits / TransferAdapter / EBBS / FxRates)
  │
  ├── Downstream returns error response
  │
  ├── Framework wraps as BusinessException / TechnicalException
  │
  ├── DomesticTransferExceptionHandler   (@RestControllerAdvice)
  │     └── @ExceptionHandler → produces ErrorResponse
  │
  ├── CustomResponseBodyAdvice           (ResponseBodyAdvice)
  │     ├── resolveErrorResponse(body)
  │     ├── resolveSource(errorResponse) → system name or "UNKNOWN"
  │     │
  │     ├── [Known source] ★ ErrorMappingService.map(systemName, errorResponse)
  │     │     ├── 1. Exact match:    systems[systemName].mappings[downstreamCode]
  │     │     ├── 2. System default: systems[systemName].system-default
  │     │     └── 3. Global fallback: defaults.fallback  (DT-X-API-201, HTTP 500)
  │     │     └── returns ResolvedMapping (expCode, title, description, httpStatus)
  │     │
  │     ├── [Unknown source] preserve original ErrorResponse code (domain/framework errors)
  │     │
  │     └── Client receives standardised ApiErrorResponse
Note on EBBS: EbbsErrorCodeMapper already maps EBBS TXNxxxxx / SYSxxxxx codes →
EbbsTransferErrorCodes (DT-X-EBBS-XXX). The new framework complements this — EBBS
entries in error-mappings.yaml act as a safety net for codes not handled by EbbsErrorCodeMapper,
referencing DT-X-EBBS-XXX codes to keep both systems consistent.

4. Execution Point in Codebase
File: handler/CustomResponseBodyAdvice.java

The ErrorMappingService is called inside beforeBodyWrite() — the single point through
which all error responses pass, regardless of which downstream system produced the error.


CustomResponseBodyAdvice.beforeBodyWrite()
    │
    ├── resolveErrorResponse(body)           ← detects ErrorResponse in body (4 shapes)
    │
    ├── resolveSource(errorResponse)         ← returns system name or "UNKNOWN"
    │
    ├── [source != UNKNOWN]
    │     ├── ★ errorMappingService.map(source, errorResponse)
    │     │         extractDownstreamCode → registry.lookup (exact → system-default → fallback)
    │     │         Returns: ResolvedMapping (expCode, title, description, httpStatus)
    │     ├── overrideHttpStatus(response, resolved.httpStatus())
    │     └── toApiErrorResponse(errorResponse, resolved)  ← DT-X-API-XXX code
    │
    ├── [source == UNKNOWN]
    │     └── toApiErrorResponse(errorResponse)            ← original code preserved
    │
    └── Client receives ApiErrorResponse
Current implementation (beforeBodyWrite — lines 138–151):


ErrorResponse errorResponse = resolveErrorResponse(body);
if (errorResponse != null) {
    String source = resolveSource(errorResponse);
    if (!SOURCE_UNKNOWN.equals(source)) {
        // Known downstream system: apply YAML-driven mapping
        ResolvedMapping resolved = errorMappingService.map(source, errorResponse);
        overrideHttpStatus(response, resolved.httpStatus());
        return toApiErrorResponse(errorResponse, resolved);
    }
    // Framework-level or unrecognised code: preserve original behaviour
    return toApiErrorResponse(errorResponse);
}
return body;
5. Error Code Namespaces
The project uses three error code enums. All exp-code values in error-mappings.yaml
must reference one of the Experience API enums (DT-X-API-XXX or DT-X-EBBS-XXX).
The downstream Deposits library codes (DP-LIB-API-XXX) and FX rates library codes
(FX-LIB-XXX) are used as mapping keys only.

TransferErrorCodes — DT-X-API-XXX (Experience API — target codes)
Range	Category
DT-X-API-001 – DT-X-API-015	Validation errors
DT-X-API-101 – DT-X-API-116	Business rule errors
DT-X-API-201 – DT-X-API-212	System / integration errors
DT-X-API-213 – DT-X-API-217	Downstream Deposits integration errors
DT-X-API-218 – DT-X-API-225	CPH downstream errors
DT-X-API-226 – DT-X-API-227	Deposits 5XX server errors
DT-X-API-228 – DT-X-API-229	FX Rates downstream errors
DT-X-API-230 – DT-X-API-231	CPH / Payee gateway server errors
EbbsTransferErrorCodes — DT-X-EBBS-XXX (Experience API — EBBS target codes)
Range	Category
DT-X-EBBS-001 – DT-X-EBBS-002	Generic / system fallback
DT-X-EBBS-003 – DT-X-EBBS-009	Transfer validation
DT-X-EBBS-010 – DT-X-EBBS-022	Account & channel
DT-X-EBBS-023 – DT-X-EBBS-045	Mandate, cheque, reversal, mobile
DepositsErrorCode — DP-LIB-API-XXX (Downstream library — mapping keys only)
From com.sc.rdpp.deposits.exception.DepositsErrorCode in payment-lib-deposits library.
These codes are thrown by the Deposits lib and used as keys in error-mappings.yaml.

Downstream Code	Enum Constant	Description
DP-LIB-API-201	CASA_ACCOUNTS_FAILED	Failed to retrieve CASA accounts information
DP-LIB-API-202	NO_CASA_ACCOUNTS_FOUND	No account information found for the given criteria
DP-LIB-API-203	ACCOUNT_BALANCE_FAILED	Failed to retrieve account balance information
DP-LIB-API-204	NO_ACCOUNT_BALANCE_FOUND	No account balance information found for the given criteria
DP-LIB-API-205	NO_TRANSFER_ACCOUNTS_FOUND	No eligible accounts found after transfer filter
DP-LIB-API-206	TRANSFER_FILTER_CONFIG_NOT_FOUND	No transfer filter configuration found for country/function code
DP-LIB-API-207	IDP_TOKEN_FAILED	Failed to retrieve IDP token for authentication
DP-LIB-API-208	CASA_ACCOUNTS_SERVER_ERROR	CASA service returned a 5XX server error while retrieving account information
DP-LIB-API-209	ACCOUNT_BALANCE_SERVER_ERROR	CASA service returned a 5XX server error while retrieving account balance
FxRateErrorCode — FX-LIB-XXX (Downstream library — mapping keys only)
From com.sc.rdpp.fxrates.exception.FxRateErrorCode in payment-lib-fx-rates library.
These codes are thrown by the FX rates lib and used as keys in error-mappings.yaml.

Downstream Code	Enum Constant	Description
FX-LIB-101	FX_RATES_LISTING_FAILED	EBBS returned a 4XX error while fetching FX rates
FX-LIB-107	FX_RATES_LISTING_SERVER_ERROR	EBBS returned a 5XX server error while fetching FX rates
6. Class Design
MappingEntryConfig.java (YAML-bindable POJO)

/**
 * YAML-bindable POJO for a single entry in error-mappings.yaml.
 * Spring Boot relaxed binding maps YAML kebab-case keys to camelCase:
 *   exp-code    → expCode
 *   http-status → httpStatus
 *
 * Convert to a runtime MappingEntry via toMappingEntry(boolean fallback).
 */
@Data
@NoArgsConstructor
public class MappingEntryConfig {
    private String expCode;   // e.g. "DT-X-API-113"
    private int httpStatus;   // e.g. 400

    public MappingEntry toMappingEntry(boolean fallback) {
        return new MappingEntry(expCode, httpStatus, fallback);
    }
}
MappingEntry.java (record)

/**
 * Immutable runtime record returned by ErrorMappingRegistry#lookup.
 * Carries only the Experience API error code and the HTTP status.
 * title and description are NOT stored here — they are resolved at runtime
 * from TransferErrorCodes / EbbsTransferErrorCodes enum by exp-code,
 * keeping the enum as the single source of truth.
 */
public record MappingEntry(
    String expCode,    // e.g. "DT-X-API-113" — must match a TransferErrorCodes constant
    int httpStatus,
    boolean isFallback // true when resolved via system default or global fallback
) {
    public MappingEntry asFallback() {
        return new MappingEntry(expCode, httpStatus, true);
    }
}
DefaultsConfig.java

/**
 * Global defaults configuration block from error-mappings.yaml.
 * Bound to the error-mapping.defaults key.
 */
@Data
@NoArgsConstructor
public class DefaultsConfig {
    private MappingEntryConfig fallback;
}
SystemMappingConfig.java

/**
 * Per-system YAML configuration block within error-mappings.yaml.
 * Spring Boot relaxed binding maps:
 *   system-default → systemDefault  (used when no exact code match is found)
 *   mappings       → mappings        (exact downstream-code to entry map)
 */
@Data
@NoArgsConstructor
public class SystemMappingConfig {
    private MappingEntryConfig systemDefault;                        // YAML key: system-default
    private Map<String, MappingEntryConfig> mappings = new HashMap<>();  // key = downstream code
}
ErrorMappingConfig.java

/**
 * Root configuration POJO for error-mappings.yaml.
 *
 * Loaded as a Spring singleton bean by ErrorMappingBeanConfig via manual YAML
 * parsing (YamlPropertiesFactoryBean + Spring Boot Binder) — NOT via
 * @ConfigurationProperties auto-scan, because the file uses a custom root key
 * (error-mapping:) separate from application.yml.
 */
@Data
@NoArgsConstructor
public class ErrorMappingConfig {
    private String version;
    private DefaultsConfig defaults;
    private Map<String, SystemMappingConfig> systems = new HashMap<>();
}
ErrorMappingRegistry.java

@Component
@RequiredArgsConstructor
public class ErrorMappingRegistry {

    private final ErrorMappingConfig config;

    public MappingEntry lookup(String system, String code) {
        SystemMappingConfig systemConfig = config.getSystems().get(system);

        if (systemConfig != null) {
            // Priority 1: Exact downstream code match
            MappingEntryConfig exactMatch = systemConfig.getMappings().get(code);
            if (exactMatch != null) {
                return exactMatch.toMappingEntry(false);
            }

            // Priority 2: System-level default
            MappingEntryConfig systemDefault = systemConfig.getSystemDefault();
            if (systemDefault != null) {
                return systemDefault.toMappingEntry(true);
            }
        }

        // Priority 3: Global fallback — DT-X-API-201 / 500
        return config.getDefaults().getFallback().toMappingEntry(true);
    }
}
ResolvedMapping.java (record)

/**
 * Fully-resolved mapping result returned by ErrorMappingService#map.
 * Carries the mapped Experience API error code together with the human-readable
 * title and description resolved from the enum at call time, and the HTTP status.
 */
public record ResolvedMapping(
    String expCode,      // e.g. "DT-X-API-113"
    String title,        // e.g. "Rule Validation failed"
    String description,  // e.g. "business.rules.configuration.failed"
    int httpStatus       // e.g. 400
) {}
ErrorMappingService.java

@Service
@RequiredArgsConstructor
public class ErrorMappingService {

    private static final Logger log = LoggerFactory.getLogger(ErrorMappingService.class);
    private static final String FALLBACK_TITLE = "Request failed";
    private static final String FALLBACK_DESCRIPTION =
            "Sorry, we are unable to process your request at this time. Please try again later.";

    private final ErrorMappingRegistry registry;

    /**
     * Maps the downstream error identified by system and the code inside errorResponse
     * to a fully-resolved ResolvedMapping.
     * Returns: expCode, title, description (from enum), httpStatus
     */
    public ResolvedMapping map(String system, ErrorResponse errorResponse) {
        String downstreamCode = extractDownstreamCode(errorResponse);
        MappingEntry entry = registry.lookup(system, downstreamCode);

        if (entry.isFallback()) {
            log.warn(APILogMarker.SYS_LOG,
                "Unmapped downstream error [system={}, code={}] — using fallback [expCode={}, httpStatus={}]",
                system, downstreamCode, entry.expCode(), entry.httpStatus());
        }

        String expCode = entry.expCode();
        return new ResolvedMapping(
            expCode,
            resolveTitle(expCode),
            resolveDescription(expCode),
            entry.httpStatus()
        );
    }

    /**
     * Extracts the most specific downstream error code from an ErrorResponse.
     *
     * For CPH: args[0] = {"errors":[{"code":"VALIDATION-003",...}]}
     *   → returns nested code "VALIDATION-003" (actual CPH business code)
     *
     * For DEPOSITS / EBBS / FXRATES: args[0] = {"downstream": responseBody}
     *   → nested extraction fails, falls back to errorResponse.code()
     *   → returns the lib's own code (DP-LIB-API-XXX, FX-LIB-XXX, etc.)
     */
    public String extractDownstreamCode(ErrorResponse errorResponse) {
        Object[] args = errorResponse.args();
        if (args != null && args.length > 0 && args[0] instanceof Map<?, ?> argsMap) {
            Object errors = argsMap.get("errors");
            if (errors instanceof List<?> list && !list.isEmpty()
                    && list.getFirst() instanceof Map<?, ?> errMap) {
                Object nestedCode = errMap.get("code");
                if (nestedCode instanceof String s && !s.isBlank()) {
                    return s;
                }
            }
        }
        // Fallback: lib-wrapped code (DP-LIB-API-XXX, FX-LIB-XXX, etc.)
        return errorResponse.code();
    }
}
7. Configuration — error-mappings.yaml
File location: src/main/resources/error-mappings.yaml

Each entry stores only exp-code and http-status.
title and description are resolved at runtime from TransferErrorCodes / EbbsTransferErrorCodes by exp-code — the enum is the single source of truth. No duplication.

Important: The per-system fallback key is system-default (not default).
This maps to SystemMappingConfig.systemDefault via Spring Boot relaxed binding.


version: "1.2"

error-mapping:
  defaults:
    fallback:
      exp-code: "DT-X-API-201"   # TransferErrorCodes.SYSTEM_ERROR
      http-status: 500

  systems:

    CPH:
      system-default:
        exp-code: "DT-X-API-206"   # CPH_GATEWAY_FAILED
        http-status: 502
      mappings:
        VALIDATION-001:
          exp-code: "DT-X-API-001"   # INVALID_REQUEST_VALUE
          http-status: 400
        VALIDATION-003:
          exp-code: "DT-X-API-113"   # BUSINESS_RULES_FAILED
          http-status: 400
        VALIDATION-004:
          exp-code: "DT-X-API-106"   # POST_DATED_TRANSFER_DATE_FAILED
          http-status: 422
        VALIDATION-010:
          exp-code: "DT-X-API-218"   # CPH_INSUFFICIENT_FUNDS
          http-status: 422
        CONFIG-001:
          exp-code: "DT-X-API-013"   # CONFIGURATION_MISSING
          http-status: 400
        CONFIG-002:
          exp-code: "DT-X-API-113"   # BUSINESS_RULES_FAILED
          http-status: 400
        # ... SCHEME-RESOLVER, DATA-COLLECTOR, IDEMPOTENCY, REGISTRY, PIPELINE, COM- ...
        IDEMPOTENCY-005:
          exp-code: "DT-X-API-219"   # IDEMPOTENCY_PREVIOUSLY_FAILED
          http-status: 409
        IDEMPOTENCY-006:
          exp-code: "DT-X-API-220"   # TRANSACTION_IN_PROGRESS
          http-status: 409
        COM-003:
          exp-code: "DT-X-API-221"   # DUPLICATE_RECORD
          http-status: 409
        CPH-BT-002:
          exp-code: "DT-X-API-222"   # CPH_TRANSACTION_TAMPERED
          http-status: 400
        CPH-BT-003:
          exp-code: "DT-X-API-223"   # CPH_RECEIPT_EXISTS
          http-status: 400
        CPH-BT-009:
          exp-code: "DT-X-API-225"   # CPH_TRANSACTION_ALREADY_SUBMITTED
          http-status: 400
        CPH-LT-013:
          exp-code: "DT-X-API-223"   # CPH_RECEIPT_EXISTS (same semantic, different HTTP)
          http-status: 422
        CPH-LT-026:
          exp-code: "DT-X-API-224"   # CPH_LIMIT_GATEWAY_FAILED
          http-status: 502
        CPH-PM-018:
          exp-code: "DT-X-API-222"   # CPH_TRANSACTION_TAMPERED
          http-status: 400

    DEPOSITS:
      system-default:
        exp-code: "DT-X-API-201"   # SYSTEM_ERROR
        http-status: 502
      mappings:
        DP-LIB-API-201:
          exp-code: "DT-X-API-213"   # CASA_ACCOUNTS_FAILED
          http-status: 400
        DP-LIB-API-202:
          exp-code: "DT-X-API-214"   # NO_CASA_ACCOUNTS_FOUND
          http-status: 404
        DP-LIB-API-203:
          exp-code: "DT-X-API-215"   # ACCOUNT_BALANCE_FAILED
          http-status: 502
        DP-LIB-API-204:
          exp-code: "DT-X-API-216"   # NO_ACCOUNT_BALANCE_FOUND
          http-status: 404
        DP-LIB-API-205:
          exp-code: "DT-X-API-217"   # NO_TRANSFER_ACCOUNTS_FOUND
          http-status: 422
        DP-LIB-API-206:
          exp-code: "DT-X-API-013"   # CONFIGURATION_MISSING (reuses existing)
          http-status: 500
        DP-LIB-API-207:
          exp-code: "DT-X-API-201"   # SYSTEM_ERROR (reuses existing)
          http-status: 500
        DP-LIB-API-208:
          exp-code: "DT-X-API-226"   # CASA_ACCOUNTS_SERVER_ERROR
          http-status: 502
        DP-LIB-API-209:
          exp-code: "DT-X-API-227"   # ACCOUNT_BALANCE_SERVER_ERROR
          http-status: 502

    EBBS:
      system-default:
        exp-code: "DT-X-EBBS-001"   # EbbsTransferErrorCodes generic fallback
        http-status: 502
      # No explicit mappings — primary EBBS handling is done by EbbsErrorCodeMapper.
      # This block acts as a safety net for any EBBS code not in EbbsErrorCodeMapper.

    FXRATES:
      system-default:
        exp-code: "DT-X-API-201"   # SYSTEM_ERROR
        http-status: 502
      mappings:
        FX-LIB-101:
          exp-code: "DT-X-API-228"   # FX_RATES_LISTING_FAILED (4XX from EBBS)
          http-status: 400
        FX-LIB-107:
          exp-code: "DT-X-API-229"   # FX_RATES_SERVER_ERROR (5XX from EBBS)
          http-status: 502
Full configuration: see src/main/resources/error-mappings.yaml

8. Configuration Storage Strategy
Loading Mechanism
error-mappings.yaml is loaded manually by ErrorMappingBeanConfig (in the config/ package)
using Spring's YamlPropertiesFactoryBean + Binder. This approach is used because the file has
a custom root key (error-mapping:) and is intentionally kept separate from application.yml.


// ErrorMappingBeanConfig.java
@Bean
public ErrorMappingConfig errorMappingConfig() {
    YamlPropertiesFactoryBean factory = new YamlPropertiesFactoryBean();
    factory.setResources(new ClassPathResource("error-mappings.yaml"));
    Properties properties = Objects.requireNonNull(factory.getObject());
    MutablePropertySources sources = new MutablePropertySources();
    sources.addFirst(new PropertiesPropertySource("error-mappings", properties));
    return new Binder(ConfigurationPropertySources.from(sources))
        .bind("error-mapping", ErrorMappingConfig.class)
        .orElseThrow();
}
The resulting bean is a singleton — parsed once at startup and cached in memory for zero-latency
access on the error path.

Deployment Options
Option	Location	When to Use
Classpath (default)	src/main/resources/error-mappings.yaml	Development; mappings rarely change
Kubernetes ConfigMap	helm/templates/error-mappings-configmap.yaml	Production; hot-reload without redeployment
Spring Cloud Config	External config repo	Enterprise-wide centralised config across services
Recommendation for this project: Start with classpath. Promote to Kubernetes ConfigMap using the existing Helm structure:


helm/
├── fk/
│   ├── sit/error-mappings-configmap.yaml
│   ├── uat/error-mappings-configmap.yaml
│   └── pt/error-mappings-configmap.yaml
└── vn/
    ├── sit/error-mappings-configmap.yaml
    └── uat/error-mappings-configmap.yaml
9. Mapping Algorithm

FUNCTION map(systemName, errorResponse) → ResolvedMapping

    // extractDownstreamCode — prefers nested args[0].errors[0].code when present
    //   CPH:                    args[0] = {errors:[{code:"VALIDATION-003"}]} → returns "VALIDATION-003"
    //   DEPOSITS / EBBS / FXRATES: args[0] = {downstream:{...}}              → falls back to errorResponse.code()
    code = extractDownstreamCode(errorResponse)

    systemConfig = config.systems[systemName]

    // Priority 1: Exact system + code match
    IF systemConfig != null AND systemConfig.mappings[code] exists:
        entry = systemConfig.mappings[code]  (isFallback=false)
        RETURN ResolvedMapping(entry.expCode, resolveTitle, resolveDescription, entry.httpStatus)

    // Priority 2: System-level default
    IF systemConfig != null AND systemConfig.systemDefault exists:
        LOG WARN "Unmapped downstream code [system={systemName}, code={code}]"
        entry = systemConfig.systemDefault  (isFallback=true)
        RETURN ResolvedMapping(entry.expCode, resolveTitle, resolveDescription, entry.httpStatus)

    // Priority 3: Global fallback — DT-X-API-201, HTTP 500
    LOG WARN "Unknown system or unmapped code [system={systemName}, code={code}]"
    RETURN ResolvedMapping(defaults.fallback.expCode, ..., defaults.fallback.httpStatus)

END FUNCTION
Lookup Priority Table
Scenario	Resolved Code	HTTP
CPH + VALIDATION-003 (exact match)	DT-X-API-113	400
CPH + UNKNOWN-999 (no exact match)	DT-X-API-206 (CPH system-default)	502
DEPOSITS + DP-LIB-API-202 (exact match)	DT-X-API-214	404
DEPOSITS + DP-LIB-API-207 (exact match)	DT-X-API-201	500
DEPOSITS + unknown DP-LIB-API-XXX	DT-X-API-201 (DEPOSITS system-default)	502
FXRATES + FX-LIB-101 (exact match)	DT-X-API-228	400
FXRATES + FX-LIB-107 (exact match)	DT-X-API-229	502
LOANS + any (unknown system)	DT-X-API-201 (global fallback)	500
UNKNOWN source (domain/framework errors)	original code preserved	original HTTP
10. Sample Mapped Output
Input — CPH VALIDATION-003

{
  "status": "0",
  "error": {
    "id": "a1b2c3d4-0000-0000-0000-000000000000",
    "code": "DT-X-API-113",
    "title": "Rule Validation failed",
    "description": "business.rules.configuration.failed",
    "args": [
      {
        "errors": [
          {
            "id": "646abc68-d576-40ed-9923-4eb41b8ad476",
            "code": "VALIDATION-003",
            "detail": "Business Rule failed: The immediate transaction transfer date must be today."
          }
        ]
      }
    ]
  },
  "currentTimestamp": "2026-06-16T11:15:54.449+0000"
}
Input — Deposits DP-LIB-API-202 (NO_CASA_ACCOUNTS_FOUND)

{
  "status": "0",
  "error": {
    "id": "b3c4d5e6-0000-0000-0000-000000000001",
    "code": "DT-X-API-214",
    "title": "No accounts found",
    "description": "No account information found for the given criteria.",
    "args": [
      {
        "downstream": {
          "errors": [
            {
              "id": "9e96ac54-a1aa-4fc7-b10b-4cf2cdcef130",
              "code": "DP-LIB-API-202",
              "detail": "No account information found for the given criteria."
            }
          ]
        }
      }
    ]
  },
  "currentTimestamp": "2026-06-17T11:15:54.449+0000"
}
11. Integration into CustomResponseBodyAdvice

// Injected via constructor
private final ErrorMappingService errorMappingService;
private static final String SOURCE_UNKNOWN = "UNKNOWN";

@Override
public Object beforeBodyWrite(Object body, ...) {

    // -- Header propagation (unchanged) --
    addHeaderIfNotNull(response, HEADER_ACCEPT_LANGUAGE, ...);
    addHeaderIfNotNull(response, HEADER_IDEMPOTENCY_KEY_DISPLAY, ...);
    addHeaderIfNotNull(response, HEADER_TRACEPARENT, ...);
    addHeaderIfNotNull(response, HEADER_X_MARKET, ...);
    addHeaderIfNotNull(response, HEADER_X_TENANT, ...);

    // -- Transform ErrorResponse → ApiErrorResponse --
    ErrorResponse errorResponse = resolveErrorResponse(body);
    if (errorResponse != null) {
        String source = resolveSource(errorResponse);
        if (!SOURCE_UNKNOWN.equals(source)) {
            // Known downstream system: apply YAML-driven mapping
            ResolvedMapping resolved = errorMappingService.map(source, errorResponse);
            overrideHttpStatus(response, resolved.httpStatus());
            return toApiErrorResponse(errorResponse, resolved);
        }
        // Framework-level or domain-validation code (e.g. DT-X-API-014, DT-X-API-001):
        // preserve original code and title via enum reverse-lookup
        return toApiErrorResponse(errorResponse);
    }

    return body;
}

/**
 * Resolves the source system name from ErrorResponse#source().
 * Set explicitly by the thrower — e.g. List.of("CPH"), List.of("DEPOSITS"), List.of("FXRATES").
 * Framework-level and domain-validation errors that do not set a source
 * return "UNKNOWN" and bypass the YAML mapping entirely.
 */
private String resolveSource(ErrorResponse errorResponse) {
    List<String> source = errorResponse.source();
    if (source != null && !source.isEmpty()) {
        return source.getFirst();
    }
    return SOURCE_UNKNOWN;
}

/**
 * Builds ApiErrorResponse using a ResolvedMapping (DT-X-API-XXX / DT-X-EBBS-XXX codes).
 * Raw downstream error is preserved in args for diagnostics.
 * Note: source field is intentionally excluded from the client response.
 */
private ApiErrorResponse toApiErrorResponse(ErrorResponse raw, ResolvedMapping resolved) {
    ErrorDetail detail = new ErrorDetail(
        raw.id(),
        resolved.expCode(),       // DT-X-API-XXX or DT-X-EBBS-XXX
        resolved.title(),
        resolved.description(),
        raw.args()                // downstream error preserved for diagnostics
    );
    return ApiErrorResponse.of(detail);
}

/**
 * Fallback overload for UNKNOWN source (framework/domain errors).
 * Preserves original code; resolves title via TransferErrorCodes then APIErrorCodes.
 */
private ApiErrorResponse toApiErrorResponse(ErrorResponse errorResponse) {
    ErrorDetail detail = new ErrorDetail(
        errorResponse.id(),
        errorResponse.code(),
        resolveTitle(errorResponse.code()),
        errorResponse.detail(),
        errorResponse.args()
    );
    return ApiErrorResponse.of(detail);
}

/**
 * Overrides the HTTP status on the response to match the mapping registry result.
 * Only applies when ServletServerHttpResponse is available.
 */
private void overrideHttpStatus(ServerHttpResponse response, int httpStatus) {
    if (response instanceof ServletServerHttpResponse servletResponse) {
        servletResponse.getServletResponse().setStatus(httpStatus);
    }
}
12. New TransferErrorCodes Entries Required
The following constants were added to TransferErrorCodes.java to cover downstream
error codes not already represented. They follow the existing DT-X-API-XXX format and range conventions.

Many CPH codes reuse existing TransferErrorCodes constants (e.g. DT-X-API-201, DT-X-API-013).
Only codes with a distinct semantic meaning that has no existing equivalent have new entries.


// -- Downstream Deposits Integration Errors (213–217) ----------------------

/** Deposits: failed to retrieve CASA accounts information (DP-LIB-API-201). */
CASA_ACCOUNTS_FAILED("DT-X-API-213", "Account retrieval failed",
    "We were unable to retrieve your account information. Please try again."),

/** Deposits: no CASA account found for the given profile criteria (DP-LIB-API-202). */
NO_CASA_ACCOUNTS_FOUND("DT-X-API-214", "No accounts found",
    "No account information found for the given criteria."),

/** Deposits: failed to retrieve account balance (DP-LIB-API-203). */
ACCOUNT_BALANCE_FAILED("DT-X-API-215", "Balance retrieval failed",
    "We were unable to retrieve your account balance. Please try again."),

/** Deposits: no account balance information found (DP-LIB-API-204). */
NO_ACCOUNT_BALANCE_FOUND("DT-X-API-216", "No balance information found",
    "No account balance information found for the given criteria."),

/** Deposits: no eligible accounts found after transfer filter (DP-LIB-API-205). */
NO_TRANSFER_ACCOUNTS_FOUND("DT-X-API-217", "No eligible accounts",
    "No eligible accounts found for this transfer. Please check your account details."),

// -- CPH Downstream Errors (218–225) ---------------------------------------

/** CPH: insufficient funds for the requested transfer (VALIDATION-010). */
CPH_INSUFFICIENT_FUNDS("DT-X-API-218", "Insufficient funds",
    "You don't have enough funds in your account to complete this transfer."),

/** CPH: request previously failed due to idempotency — cannot be retried (IDEMPOTENCY-005). */
IDEMPOTENCY_PREVIOUSLY_FAILED("DT-X-API-219", "Request previously failed",
    "This request has previously failed and cannot be retried. Please submit a new request."),

/** CPH: a transaction with this idempotency key is currently in progress (IDEMPOTENCY-006). */
TRANSACTION_IN_PROGRESS("DT-X-API-220", "Transaction in progress",
    "A transaction for this request is currently being processed. Please wait and try again."),

/** CPH: a record for this request already exists (COM-003). */
DUPLICATE_RECORD("DT-X-API-221", "Duplicate record",
    "A record for this request already exists. Please check your request and try again."),

/** CPH: transaction details have been tampered (CPH-BT-002, CPH-PM-018). */
CPH_TRANSACTION_TAMPERED("DT-X-API-222", "Transaction integrity failed",
    "The transaction could not be processed due to a data integrity issue. Please try again."),

/** CPH: receipt number already exists (CPH-BT-003, CPH-LT-013). */
CPH_RECEIPT_EXISTS("DT-X-API-223", "Duplicate receipt",
    "A transaction with this receipt number already exists."),

/** CPH: unexpected error in the transaction limit gateway (CPH-LT-026). */
CPH_LIMIT_GATEWAY_FAILED("DT-X-API-224", "Transaction limit gateway failed",
    "Sorry, we are unable to process your request at this time. Please try again later."),

/** CPH: transaction has already been submitted (CPH-BT-009). */
CPH_TRANSACTION_ALREADY_SUBMITTED("DT-X-API-225", "Transaction already submitted",
    "This transaction has already been submitted and cannot be resubmitted."),

// -- Deposits 5XX Server Error Codes (226–227) -----------------------------

/** Deposits: CASA service returned a 5XX server error retrieving account info (DP-LIB-API-208). */
CASA_ACCOUNTS_SERVER_ERROR("DT-X-API-226", "CASA service error",
    "CASA service returned a server error. Please try again later."),

/** Deposits: CASA service returned a 5XX server error retrieving account balance (DP-LIB-API-209). */
ACCOUNT_BALANCE_SERVER_ERROR("DT-X-API-227", "Account balance service error",
    "Failed to retrieve account balance due to a CASA service error. Please try again later."),

// -- FX Rates Downstream Errors (228–229) ----------------------------------

/** FX Rates: EBBS returned a 4XX error while fetching rates (FX-LIB-101). */
FX_RATES_LISTING_FAILED("DT-X-API-228", "FX rates retrieval failed",
    "We were unable to retrieve FX rates. Please try again."),

/** FX Rates: EBBS returned a 5XX server error while fetching rates (FX-LIB-107). */
FX_RATES_SERVER_ERROR("DT-X-API-229", "FX rates service error",
    "FX rates service returned a server error. Please try again later."),

// -- CPH / Payee Gateway Server Errors (230–231) ---------------------------

/** CPH gateway returned a 5XX server error; parameter: responseBody. */
CPH_GATEWAY_SERVER_ERROR("DT-X-API-230", "CPH gateway server error",
    "CPH service returned a server error: %s"),

/** Payee gateway returned a 5XX server error; parameter: responseBody. */
PAYEE_GATEWAY_SERVER_ERROR("DT-X-API-231", "Payee gateway server error",
    "Payee service returned a server error: %s");
Mapping Summary — All New Codes
Deposits (DP-LIB-API-XXX)
Downstream Code	Downstream Constant	DT-X-API-XXX	Reuses Existing?
DP-LIB-API-201	CASA_ACCOUNTS_FAILED	DT-X-API-213	No — new
DP-LIB-API-202	NO_CASA_ACCOUNTS_FOUND	DT-X-API-214	No — new
DP-LIB-API-203	ACCOUNT_BALANCE_FAILED	DT-X-API-215	No — new
DP-LIB-API-204	NO_ACCOUNT_BALANCE_FOUND	DT-X-API-216	No — new
DP-LIB-API-205	NO_TRANSFER_ACCOUNTS_FOUND	DT-X-API-217	No — new
DP-LIB-API-206	TRANSFER_FILTER_CONFIG_NOT_FOUND	DT-X-API-013	✅ Yes — CONFIGURATION_MISSING
DP-LIB-API-207	IDP_TOKEN_FAILED	DT-X-API-201	✅ Yes — SYSTEM_ERROR
DP-LIB-API-208	CASA_ACCOUNTS_SERVER_ERROR	DT-X-API-226	No — new
DP-LIB-API-209	ACCOUNT_BALANCE_SERVER_ERROR	DT-X-API-227	No — new
CPH (multiple prefix families)
Downstream Code	Description	DT-X-API-XXX	Reuses Existing?
VALIDATION-001	Structural validation failed	DT-X-API-001	✅ Yes — INVALID_REQUEST_VALUE
VALIDATION-003	Business rule failed	DT-X-API-113	✅ Yes — BUSINESS_RULES_FAILED
VALIDATION-004	Post-dated date invalid	DT-X-API-106	✅ Yes — POST_DATED_TRANSFER_DATE_FAILED
VALIDATION-010	Insufficient funds	DT-X-API-218	No — new
CONFIG-001	Service config not found	DT-X-API-013	✅ Yes — CONFIGURATION_MISSING
CONFIG-002	Business rules not found	DT-X-API-113	✅ Yes — BUSINESS_RULES_FAILED
SCHEME-RESOLVER-001	Scheme generation failed	DT-X-API-201	✅ Yes — SYSTEM_ERROR
DATA-COLLECTOR-002	Data collection failed	DT-X-API-202	✅ Yes — NO_DATA_COLLECTOR_FOUND
IDEMPOTENCY-001	No idempotency record found	DT-X-API-201	✅ Yes — SYSTEM_ERROR
IDEMPOTENCY-002	Lock acquisition error	DT-X-API-201	✅ Yes — SYSTEM_ERROR
IDEMPOTENCY-003	Pipeline execution error	DT-X-API-201	✅ Yes — SYSTEM_ERROR
IDEMPOTENCY-004	Deserialize cached response failed	DT-X-API-201	✅ Yes — SYSTEM_ERROR
IDEMPOTENCY-005	Previously failed — cannot retry	DT-X-API-219	No — new
IDEMPOTENCY-006	Transaction currently in progress	DT-X-API-220	No — new
REGISTRY-001	Registry not found	DT-X-API-201	✅ Yes — SYSTEM_ERROR
REGISTRY-002	Registry already exists	DT-X-API-201	✅ Yes — SYSTEM_ERROR
PIPELINE-001	No strategy found	DT-X-API-201	✅ Yes — SYSTEM_ERROR
PIPELINE-002	Unsupported JSON:API operator	DT-X-API-201	✅ Yes — SYSTEM_ERROR
COM-001	Database error	DT-X-API-201	✅ Yes — SYSTEM_ERROR
COM-003	Record already exists	DT-X-API-221	No — new
COM-004	Internal error	DT-X-API-201	✅ Yes — SYSTEM_ERROR
COM-006	Request body missing/invalid	DT-X-API-001	✅ Yes — INVALID_REQUEST_VALUE
CPH-BT-002	Transaction details tampered	DT-X-API-222	No — new
CPH-BT-003	Receipt number already exists (400)	DT-X-API-223	No — new
CPH-BT-004	TxnCurr must be DbtAccCurr or CdtAccCur	DT-X-API-104	✅ Yes — TXN_CURR_FAILED
CPH-BT-009	Transaction already submitted	DT-X-API-225	No — new
CPH-LT-001	ConstraintViolation (field-level)	DT-X-API-001	✅ Yes — INVALID_REQUEST_VALUE
CPH-LT-002	Pipeline strategy not found	DT-X-API-201	✅ Yes — SYSTEM_ERROR
CPH-LT-003	Source/dest same OR invalid format	DT-X-API-001	✅ Yes — INVALID_REQUEST_VALUE
CPH-LT-004	Currency mismatch	DT-X-API-104	✅ Yes — TXN_CURR_FAILED
CPH-LT-010	Transaction not found	DT-X-API-205	✅ Yes — NO_PAYMENT_FOUND
CPH-LT-011	Unexpected error in CASA gateway	DT-X-API-201	✅ Yes — SYSTEM_ERROR
CPH-LT-013	Receipt already exists (422)	DT-X-API-223	✅ Yes — CPH_RECEIPT_EXISTS (diff HTTP)
CPH-LT-014	Minimum limit not provided	DT-X-API-101	✅ Yes — MINIMUM_AMOUNT_RULE_FAILED
CPH-LT-026	Transaction limit gateway failed	DT-X-API-224	No — new
CPH-LT-034	Amount exceeded transfer limit	DT-X-API-102	✅ Yes — MAXIMUM_AMOUNT_RULE_FAILED
CPH-PM-018	Transaction details tampered	DT-X-API-222	✅ Yes — CPH_TRANSACTION_TAMPERED
FXRATES (FX-LIB-XXX)
Downstream Code	Downstream Constant	DT-X-API-XXX	Reuses Existing?
FX-LIB-101	FX_RATES_LISTING_FAILED	DT-X-API-228	No — new
FX-LIB-107	FX_RATES_LISTING_SERVER_ERROR	DT-X-API-229	No — new
13. File Inventory
New Files
File	Package	Purpose
MappingEntryConfig.java	handler.exception.mapping	YAML-bindable POJO for a single mapping entry; converts to MappingEntry via toMappingEntry()
MappingEntry.java	handler.exception.mapping	Immutable runtime record for a resolved mapping entry
DefaultsConfig.java	handler.exception.mapping	POJO for the defaults.fallback block in the YAML
SystemMappingConfig.java	handler.exception.mapping	POJO for a per-system YAML config block (system-default + mappings)
ErrorMappingConfig.java	handler.exception.mapping	Root config POJO (version, defaults, systems)
ErrorMappingRegistry.java	handler.exception.mapping	Caches config; exposes lookup(system, code) with 3-level priority
ResolvedMapping.java	handler.exception.mapping	Fully-resolved record returned by ErrorMappingService#map (expCode, title, description, httpStatus)
ErrorMappingService.java	handler.exception.mapping	Lookup orchestration + unmapped warning logging; returns ResolvedMapping
ErrorMappingBeanConfig.java	config	Loads error-mappings.yaml via YamlPropertiesFactoryBean + Binder; registers ErrorMappingConfig bean
error-mappings.yaml	src/main/resources/	Mapping configuration — source of truth for all downstream → experience code mappings
Modified Files
File	Change
TransferErrorCodes.java	Added DT-X-API-213–DT-X-API-231 (5 Deposits + 8 CPH + 2 Deposits 5XX + 2 FX Rates + 2 Gateway Server Errors)
CustomResponseBodyAdvice.java	Injected ErrorMappingService; added resolveSource(), overrideHttpStatus(); two toApiErrorResponse() overloads (mapped vs. UNKNOWN)
DOCUMENTATION_INDEX.md	Added entry under Feature Design Documents
14. Best Practices & Pitfalls
✅ Do
 Always provide a system-default per system and a global fallback — no error should be silent
 Log every unmapped code with APILogMarker.SYS_LOG — consistent with project standards
 Keep raw downstream detail in args only — never in client-facing description
 Use manual YamlPropertiesFactoryBean loading — parsed once at startup, cached in memory, never per-request I/O
 Unit test every known mapping: assert expCode and httpStatus per downstream code
 Version the YAML (version: "1.x") for audit and change tracking
 Add new DT-X-API-XXX enum entries to TransferErrorCodes for every new downstream semantic
❌ Avoid
 Hardcoding downstream codes in Java logic — config only
 Exposing raw detail from downstream as the client description
 Removing existing mapping entries — deprecate with a comment instead
 Reusing the same DT-X-API-XXX code for different semantic meanings
 Silently swallowing unmapped codes without logging
 Using default: as the YAML key for system fallback — the correct key is system-default:
Governance
Concern	Recommendation
Ownership	API Platform / Integration Team owns error-mappings.yaml
New downstream codes	Downstream teams raise a PR to add new entries; reviewed before merge
Backward compatibility	Never remove — mark obsolete with # deprecated comment
Runtime reload	@RefreshScope + Kubernetes ConfigMap for zero-downtime updates
Observability	Alert when error_mapping.unmapped rate exceeds threshold
15. Implementation Checklist
 Create handler/exception/mapping/ package
 Create MappingEntryConfig.java YAML-bindable POJO with toMappingEntry()
 Create MappingEntry.java record with asFallback() helper
 Create DefaultsConfig.java POJO
 Create SystemMappingConfig.java POJO (field: systemDefault, YAML key: system-default)
 Create ErrorMappingConfig.java (plain POJO — loaded via ErrorMappingBeanConfig, not @ConfigurationProperties)
 Create ErrorMappingBeanConfig.java in config/ — YamlPropertiesFactoryBean + Binder loading
 Create ErrorMappingRegistry.java with 3-level lookup (exact → system-default → global fallback)
 Create ResolvedMapping.java record
 Create ErrorMappingService.java — returns ResolvedMapping; APILogMarker.SYS_LOG warn on unmapped
 Create src/main/resources/error-mappings.yaml with CPH, DEPOSITS, EBBS, FXRATES systems
 Add DT-X-API-213 to DT-X-API-231 to TransferErrorCodes.java
 Modify CustomResponseBodyAdvice.java — inject service; resolveSource(); overrideHttpStatus(); two toApiErrorResponse() overloads
 Write unit tests for ErrorMappingRegistry (exact match, system-default, global fallback)
 Write unit tests for ErrorMappingService (mapped, unmapped, unknown system, FXRATES)
 Write integration test — end-to-end through CustomResponseBodyAdvice
 Update DOCUMENTATION_INDEX.md to reference this document
 Code review
 Deploy to SIT
Error Mapping Design v1.6 | Last Updated: 26 June 2026

v1.6 changes: corrected SystemMappingConfig field (systemDefault / YAML key system-default:);
removed @ConfigurationProperties from ErrorMappingConfig — documented ErrorMappingBeanConfig manual loading;
fixed ErrorMappingRegistry stub (getSystemDefault() not getDefaultEntry());
fixed ErrorMappingService.map() return type to ResolvedMapping;
added MappingEntryConfig, DefaultsConfig, ResolvedMapping, ErrorMappingBeanConfig to class design and file inventory;
added FXRATES system (FX-LIB-101, FX-LIB-107) and DP-LIB-API-208/209 throughout;
fixed §9 lookup example (VALIDATION-003 → DT-X-API-113 HTTP 400);
updated beforeBodyWrite stub with SOURCE_UNKNOWN branch and overrideHttpStatus();
added DT-X-API-226–DT-X-API-231 to §12.
