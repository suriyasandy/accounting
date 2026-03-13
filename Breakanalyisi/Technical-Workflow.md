# Technical Workflow Architecture Document
## LLM-Assisted Accounting Control Breaks Resolution System

**Document Version:** 1.0  
**Date:** March 13, 2026  
**Owner:** Market Middle Office - OMRC AI/ML Team  
**Classification:** Internal Technical Specification

---

## 1. System Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DATA LAYER (Source Systems)                      │
├─────────────────────────────────────────────────────────────────────┤
│  • Reconciliation Cubes (621K breaks/month)                         │
│  • JIRA API (Issue tracking, EPIC linking)                          │
│  • Outlook/Exchange (Email extraction - future)                     │
│  • Historical Break Database (24-month resolved breaks)             │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   INGESTION & PREPROCESSING                          │
├─────────────────────────────────────────────────────────────────────┤
│  • Data extraction pipeline (Python/Pandas)                         │
│  • Field validation & type conversion                               │
│  • Deduplication (TRADE REF)                                        │
│  • True/Systemic break classification (CatBoost)                    │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      ML MODELS LAYER                                │
├─────────────────────────────────────────────────────────────────────┤
│  ┌───────────────┐  ┌───────────────┐  ┌────────────────┐          │
│  │ CatBoost      │  │ CatBoost      │  │ CatBoost       │          │
│  │ Severity      │  │ ISSUE         │  │ RAG Rating     │          │
│  │ Scorer        │  │ CATEGORY      │  │ Predictor      │          │
│  └───────────────┘  └───────────────┘  └────────────────┘          │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    RAG & LLM LAYER                                  │
├─────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Vector Store (ChromaDB / FAISS)                            │    │
│  │  • 14.9M embedded historical breaks                         │    │
│  │  • Embedding model: text-embedding-3-small                  │    │
│  │  • Chunk size: 400-600 tokens (Comments + JIRA DESC)        │    │
│  └────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Azure OpenAI GPT-4o-mini                                   │    │
│  │  • Input: Break fields + RAG context + system prompt        │    │
│  │  • Output: Root cause + Action + Confidence score           │    │
│  │  • Caching: 50% hit rate (prompt caching)                   │    │
│  │  • Rate limit: 10K tokens/min                               │    │
│  └────────────────────────────────────────────────────────────┘    │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   AGENTIC AI ORCHESTRATION                          │
│                        (LangGraph)                                   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│  │ Triage   │──→│   RCA    │──→│  JIRA    │──→│ Reporting│        │
│  │ Agent    │   │  Agent   │   │  Agent   │   │  Agent   │        │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘        │
│       │              │               │               │              │
│       ├──→ Auto-Resolve Agent (Systematic)                          │
│       └──→ Escalation Agent (ABS GBP >1mn, Age >30d)               │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│              HUMAN-IN-THE-LOOP UI (Streamlit)                       │
├─────────────────────────────────────────────────────────────────────┤
│  • Break Review Dashboard                                           │
│  • LLM Suggestion Display (Root Cause + Action + Confidence)       │
│  • Approval Workflow (ACCEPT / MODIFY / REJECT buttons)            │
│  • Historical RAG sources citation                                  │
│  • Audit trail viewer                                               │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   OUTPUT & INTEGRATION                              │
├─────────────────────────────────────────────────────────────────────┤
│  • Update break dataset (Root Cause identified, B/S Cert)          │
│  • JIRA ticket creation/update                                      │
│  • Daily digest report (email to entity leads)                      │
│  • Governance dashboard (acceptance rates, KPIs)                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Data Flow: Break Processing Pipeline

### 2.1 Input Processing

**Step 1: Break Ingestion**
```python
# Extract breaks from reconciliation cube
breaks_df = pd.read_csv('monthly_breaks_YYYYMM.csv')

# Validate schema (41 columns)
required_cols = ['Period', 'Asset Class', 'TRADE REF', 'Type of break', 
                 'BREAK AMOUNT GBP', 'Comments', 'JIRA DESC', 'Age Days']
validate_columns(breaks_df, required_cols)

# Data type conversion
breaks_df['BREAK AMOUNT GBP'] = pd.to_numeric(breaks_df['BREAK AMOUNT GBP'])
breaks_df['Age Days'] = pd.to_numeric(breaks_df['Age Days'])
```

**Step 2: Deduplication**
```python
# Remove duplicates by TRADE REF
breaks_df = breaks_df.drop_duplicates(subset=['TRADE REF'], keep='last')
```

**Step 3: True/Systemic Classification (CatBoost)**
```python
from catboost import CatBoostClassifier

# Load pre-trained model
model_ts = CatBoostClassifier()
model_ts.load_model('models/true_systemic_classifier_v1.cbm')

# Predict
cat_features = ['Asset Class', 'Type of break', 'Account Group', 
                'Team', 'Entity', 'Bucket']
breaks_df['is_systemic_pred'] = model_ts.predict(
    breaks_df[cat_features + numeric_features]
)

# Filter to TRUE breaks for LLM processing
true_breaks = breaks_df[breaks_df['is_systemic_pred'] == 0]
```

---

### 2.2 RAG Vector Store Query

**Embedding Generation**
```python
from sentence_transformers import SentenceTransformer
import faiss

# Load embedding model
embed_model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')

# Prepare query text
def prepare_query_text(row):
    return f"{row['Comments']} {row['JIRA DESC']} {row['EPIC DESC']}"

query_text = prepare_query_text(break_row)
query_embedding = embed_model.encode([query_text])

# FAISS vector search
index = faiss.read_index('vector_stores/historical_breaks.index')
D, I = index.search(query_embedding.astype('float32'), k=5)

# Retrieve top-5 most similar historical breaks
similar_breaks = historical_df.iloc[I[0]]
```

**RAG Context Construction**
```python
def build_rag_context(similar_breaks):
    context = "Similar historical breaks:\n\n"
    for idx, (_, row) in enumerate(similar_breaks.iterrows(), 1):
        context += f"{idx}. Break ID: {row['TRADE REF']}\n"
        context += f"   Root Cause: {row['Root Cause identified']}\n"
        context += f"   Action Taken: {row['Action']}\n"
        context += f"   Comments: {row['Comments'][:200]}...\n\n"
    return context
```

---

### 2.3 LLM Invocation (Azure OpenAI)

**Prompt Template**
```python
SYSTEM_PROMPT = """You are a financial reconciliation expert analyzing accounting control breaks.
Your task is to determine the root cause category from the provided taxonomy and suggest a resolution action.

Root Cause Taxonomy (use EXACTLY these labels):
1. Timing Mismatch
2. Price Discrepancy
3. Quantity Mismatch
4. Settlement Instruction (SSI) Error
5. Trade Booking Error
6. FX Rate Difference
7. Fee/Commission Mismatch
8. Corporate Action Impact
9. System Outage/Technical Issue
10. Missing Trade
11. Duplicate Trade
12. Other (Manual Review Required)

CRITICAL RULES:
- Output ONLY the root cause label from the taxonomy above
- If uncertain, output "Other (Manual Review Required)"
- Do NOT invent new categories
- Base your reasoning on the similar historical breaks provided
"""

USER_PROMPT_TEMPLATE = """
Current Break Details:
- Trade Reference: {trade_ref}
- Asset Class: {asset_class}
- Type of Break: {type_of_break}
- Break Amount (GBP): {break_amount_gbp}
- Age Days: {age_days}
- Comments: {comments}
- JIRA Description: {jira_desc}

{rag_context}

Based on the above, determine:
1. Root Cause Category (from taxonomy)
2. Suggested Resolution Action (based on similar historical actions)
3. Confidence Score (0-100)

Output format (JSON):
{{
  "root_cause": "<category from taxonomy>",
  "action": "<suggested resolution steps>",
  "confidence": <0-100>,
  "supporting_break_ids": ["<ref1>", "<ref2>", "<ref3>"]
}}
"""
```

**API Call**
```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    api_version="2024-02-01",
    azure_endpoint="https://<your-resource>.openai.azure.com"
)

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": user_prompt}
    ],
    temperature=0.2,  # Low temp for consistency
    max_tokens=300,
    response_format={"type": "json_object"}
)

llm_output = json.loads(response.choices[0].message.content)
```

---

### 2.4 Agentic Workflow (LangGraph)

**State Definition**
```python
from typing import TypedDict
from langgraph.graph import StateGraph, END

class BreakState(TypedDict):
    break_id: str
    break_data: dict
    severity_score: float
    is_systemic: bool
    issue_category: str
    rag_rating: str
    llm_root_cause: str
    llm_action: str
    llm_confidence: float
    jira_ref: str
    escalated: bool
    final_status: str
```

**Agent Nodes**

*Triage Agent*
```python
def triage_agent(state: BreakState):
    """Classify break and route to appropriate agent."""
    break_data = state['break_data']
    
    # CatBoost predictions
    severity = severity_model.predict(break_data)[0]
    is_systemic = systemic_model.predict(break_data)[0]
    issue_cat = category_model.predict(break_data)[0]
    rag = rag_model.predict(break_data)[0]
    
    return {
        **state,
        'severity_score': severity,
        'is_systemic': bool(is_systemic),
        'issue_category': issue_cat,
        'rag_rating': rag
    }
```

*RCA Agent*
```python
def rca_agent(state: BreakState):
    """LLM-based root cause analysis."""
    break_data = state['break_data']
    
    # RAG retrieval
    similar_breaks = query_vector_store(break_data)
    rag_context = build_rag_context(similar_breaks)
    
    # LLM call
    llm_output = call_azure_openai(break_data, rag_context)
    
    return {
        **state,
        'llm_root_cause': llm_output['root_cause'],
        'llm_action': llm_output['action'],
        'llm_confidence': llm_output['confidence']
    }
```

*Auto-Resolve Agent*
```python
def auto_resolve_agent(state: BreakState):
    """Auto-resolve systematic breaks with known fixes."""
    if state['is_systemic'] and state['llm_confidence'] > 90:
        # Execute pre-approved systematic resolution
        post_journal_entry(state['break_data'])
        update_jira_status(state['jira_ref'], "Resolved - Systematic")
        return {**state, 'final_status': 'Auto-Resolved'}
    else:
        return {**state, 'final_status': 'Manual Review Required'}
```

*Escalation Agent*
```python
def escalation_agent(state: BreakState):
    """Escalate high-value or aged breaks."""
    break_data = state['break_data']
    
    should_escalate = (
        break_data['ABS GBP'] > 1_000_000 or
        break_data['Age Days'] > 30 or
        state['rag_rating'] == 'RED'
    )
    
    if should_escalate:
        send_alert_email(
            to=break_data['Entity'] + '_lead@firm.com',
            subject=f"ESCALATION: Break {state['break_id']}",
            body=f"High priority break requiring attention:\n"
                 f"Amount: £{break_data['ABS GBP']:,.2f}\n"
                 f"Age: {break_data['Age Days']} days\n"
                 f"Root Cause (LLM): {state['llm_root_cause']}"
        )
        return {**state, 'escalated': True}
    
    return {**state, 'escalated': False}
```

**Routing Logic**
```python
def route_break(state: BreakState):
    """Conditional routing based on break characteristics."""
    if state['is_systemic'] and state['severity_score'] < 30:
        return "auto_resolve"
    elif state['severity_score'] > 80 or state['break_data']['ABS GBP'] > 1_000_000:
        return "escalate"
    else:
        return "rca_agent"
```

**Graph Construction**
```python
graph = StateGraph(BreakState)

# Add nodes
graph.add_node("triage", triage_agent)
graph.add_node("rca_agent", rca_agent)
graph.add_node("auto_resolve", auto_resolve_agent)
graph.add_node("escalate", escalation_agent)
graph.add_node("jira_agent", jira_update_agent)
graph.add_node("reporting", reporting_agent)

# Entry point
graph.set_entry_point("triage")

# Conditional edges from triage
graph.add_conditional_edges(
    "triage",
    route_break,
    {
        "auto_resolve": "auto_resolve",
        "escalate": "escalate",
        "rca_agent": "rca_agent"
    }
)

# All paths converge to JIRA + Reporting
for node in ["rca_agent", "auto_resolve", "escalate"]:
    graph.add_edge(node, "jira_agent")

graph.add_edge("jira_agent", "reporting")
graph.add_edge("reporting", END)

# Compile
app = graph.compile()
```

---

## 3. Human-in-the-Loop Interface (Streamlit)

### 3.1 Dashboard Layout

**Page 1: Break Review Queue**
```python
import streamlit as st

st.title("🔍 Accounting Control Breaks - AI Review Dashboard")

# Filters
col1, col2, col3 = st.columns(3)
with col1:
    entity_filter = st.multiselect("Entity", breaks_df['Entity'].unique())
with col2:
    asset_class_filter = st.multiselect("Asset Class", breaks_df['Asset Class'].unique())
with col3:
    confidence_threshold = st.slider("Min LLM Confidence", 0, 100, 70)

# Display breaks requiring review
pending_breaks = breaks_df[
    (breaks_df['llm_confidence'] >= confidence_threshold) &
    (breaks_df['analyst_decision'].isna())
]

st.metric("Breaks Pending Review", len(pending_breaks))

# Break cards
for idx, row in pending_breaks.iterrows():
    with st.expander(f"Break {row['TRADE REF']} - £{row['ABS GBP']:,.2f}"):
        col_left, col_right = st.columns([2, 1])
        
        with col_left:
            st.markdown("**Break Details:**")
            st.write(f"Asset Class: {row['Asset Class']}")
            st.write(f"Type: {row['Type of break']}")
            st.write(f"Age: {row['Age Days']} days")
            st.write(f"Comments: {row['Comments']}")
            
            st.markdown("---")
            st.markdown("**🤖 AI Suggestion:**")
            st.success(f"Root Cause: {row['llm_root_cause']}")
            st.info(f"Action: {row['llm_action']}")
            st.metric("Confidence", f"{row['llm_confidence']}%")
            
            # Show RAG sources
            with st.expander("📚 Supporting Evidence (RAG)"):
                for source_id in row['supporting_break_ids']:
                    source_break = historical_df[historical_df['TRADE REF'] == source_id].iloc[0]
                    st.markdown(f"**{source_id}** - {source_break['Root Cause identified']}")
                    st.caption(source_break['Action'])
        
        with col_right:
            st.markdown("**Analyst Decision:**")
            
            # Approval workflow
            decision = st.radio(
                "Action",
                ["ACCEPT", "MODIFY", "REJECT"],
                key=f"decision_{row['TRADE REF']}"
            )
            
            if decision == "MODIFY":
                modified_root_cause = st.text_input(
                    "Correct Root Cause",
                    value=row['llm_root_cause'],
                    key=f"root_{row['TRADE REF']}"
                )
                modified_action = st.text_area(
                    "Correct Action",
                    value=row['llm_action'],
                    key=f"action_{row['TRADE REF']}"
                )
            elif decision == "REJECT":
                rejection_reason = st.text_area(
                    "Rejection Reason",
                    key=f"reject_{row['TRADE REF']}"
                )
            
            if st.button("Submit", key=f"submit_{row['TRADE REF']}"):
                # Log decision
                log_analyst_decision(
                    break_id=row['TRADE REF'],
                    analyst_id=st.session_state['user_id'],
                    decision=decision,
                    final_root_cause=modified_root_cause if decision == "MODIFY" else row['llm_root_cause'],
                    final_action=modified_action if decision == "MODIFY" else row['llm_action'],
                    timestamp=datetime.now()
                )
                
                # Update database
                update_break_record(row['TRADE REF'], ...)
                
                st.success("✅ Decision recorded!")
                st.rerun()
```

**Page 2: Governance Dashboard**
```python
st.title("📊 AI Governance Dashboard")

# Date range
date_range = st.date_input("Analysis Period", [start_date, end_date])

# KPIs
col1, col2, col3, col4 = st.columns(4)
with col1:
    acceptance_rate = calculate_acceptance_rate(date_range)
    st.metric("Acceptance Rate", f"{acceptance_rate:.1%}", 
              delta=f"{acceptance_rate - 0.85:.1%} vs target")
with col2:
    avg_confidence = calculate_avg_confidence(date_range)
    st.metric("Avg LLM Confidence", f"{avg_confidence:.1f}%")
with col3:
    time_saved = calculate_time_saved(date_range)
    st.metric("Analyst Hours Saved", f"{time_saved:,.0f}")
with col4:
    population_rate = calculate_field_population(date_range)
    st.metric("Root Cause Population", f"{population_rate:.1%}")

# Acceptance by Root Cause Category
st.markdown("### Acceptance Rate by Root Cause")
acceptance_by_cat = breaks_df.groupby('llm_root_cause').agg({
    'analyst_decision': lambda x: (x == 'ACCEPT').mean()
}).reset_index()

st.bar_chart(acceptance_by_cat.set_index('llm_root_cause'))

# Override analysis
st.markdown("### Top Analyst Modifications")
modifications = breaks_df[breaks_df['analyst_decision'] == 'MODIFY']
st.dataframe(modifications[['TRADE REF', 'llm_root_cause', 'final_root_cause', 'analyst_id']])
```

---

## 4. Infrastructure & Deployment

### 4.1 Technology Stack

| Component | Technology | Justification |
|-----------|------------|---------------|
| **ML Models** | CatBoost 1.2+ | Best-in-class for tabular data; handles categorical features natively |
| **LLM** | Azure OpenAI GPT-4o-mini | Enterprise deployment; data residency; competitive pricing |
| **Embedding** | text-embedding-3-small | Cost-effective; 1536 dimensions; excellent for financial text |
| **Vector DB** | ChromaDB / FAISS | Open-source; easy Python integration; supports metadata filtering |
| **Orchestration** | LangGraph 0.2+ | Stateful multi-agent workflows; graph-based logic |
| **Backend** | Python 3.11+, FastAPI | Async API; type hints; OpenAPI auto-docs |
| **Frontend** | Streamlit 1.35+ | Rapid prototyping; built-in auth; easy ML integration |
| **Database** | PostgreSQL 16 | ACID compliance; JSONB support; full-text search |
| **Message Queue** | Redis 7+ | Caching; rate limiting; session management |
| **Monitoring** | Prometheus + Grafana | Time-series metrics; custom dashboards |
| **Logging** | ELK Stack (Elasticsearch, Logstash, Kibana) | Centralized logging; full-text search; audit trail |

### 4.2 Azure Infrastructure

```yaml
Resource Group: rg-omrc-ai-prod

Resources:
  - Azure OpenAI Service (GPT-4o-mini, text-embedding-3-small)
    Region: UK South
    Private Endpoint: Enabled
    Managed Identity: Enabled
  
  - Azure Kubernetes Service (AKS)
    Node Count: 3
    VM Size: Standard_D4s_v3
    Auto-scaling: 3-10 nodes
  
  - Azure Database for PostgreSQL
    Tier: General Purpose
    vCores: 4
    Storage: 512 GB
    Backup Retention: 30 days
  
  - Azure Cache for Redis
    Tier: Premium P1
    Size: 6 GB
  
  - Azure Virtual Network
    Address Space: 10.0.0.0/16
    Subnets:
      - aks-subnet: 10.0.1.0/24
      - db-subnet: 10.0.2.0/24
      - openai-subnet: 10.0.3.0/24
  
  - Azure Monitor
    Log Analytics Workspace
    Application Insights
  
  - Azure Key Vault
    Secrets: API keys, DB credentials
```

### 4.3 Security Architecture

**Network Security**
- Private endpoints for all Azure services (no public internet access)
- VNet integration for AKS
- Network Security Groups (NSGs) with least-privilege rules
- Azure Firewall for egress filtering

**Identity & Access**
- Azure AD authentication for Streamlit UI (SSO)
- Managed Service Identity (MSI) for service-to-service auth
- RBAC for resource access (Contributor, Reader roles)
- MFA required for all human access

**Data Protection**
- TLS 1.3 for all in-transit data
- AES-256 encryption at rest (Azure Storage encryption)
- Column-level encryption for sensitive fields
- Data retention: 7 years (regulatory requirement)

**Audit & Compliance**
- All LLM API calls logged to Azure Monitor
- Immutable audit log (append-only)
- Weekly security scans (Azure Defender)
- Quarterly penetration testing

---

## 5. Model Validation & Testing (SR 11-7)

### 5.1 Validation Framework

**Phase 1: Development Validation**
```python
# Backtesting on historical data
historical_labeled = breaks_df[breaks_df['Root Cause identified'].notna()]

# Train/test split (80/20, time-based)
train_cutoff = '2025-06-01'
train = historical_labeled[historical_labeled['Date'] < train_cutoff]
test = historical_labeled[historical_labeled['Date'] >= train_cutoff]

# Generate LLM predictions on test set
test['llm_pred_root_cause'] = test.apply(lambda row: llm_predict(row), axis=1)

# Accuracy metrics
from sklearn.metrics import classification_report, confusion_matrix

print(classification_report(
    test['Root Cause identified'],
    test['llm_pred_root_cause'],
    target_names=ROOT_CAUSE_TAXONOMY
))

# Confusion matrix by category
cm = confusion_matrix(test['Root Cause identified'], test['llm_pred_root_cause'])
plot_confusion_matrix(cm, ROOT_CAUSE_TAXONOMY)
```

**Phase 2: Independent Validation (Model Risk Team)**
- **Conceptual soundness review:** Prompt template, taxonomy design, RAG retrieval logic
- **Input data quality:** Historical break completeness, label accuracy
- **Performance testing:** Precision/recall per category; edge case handling
- **Sensitivity analysis:** Performance degradation with missing fields

**Phase 3: Pilot Validation**
- Parallel run: LLM suggestions vs manual analyst labels
- Inter-rater reliability: Compare LLM vs analyst agreement rates
- Real-world performance: Acceptance rate, modification patterns

### 5.2 Testing Strategy

| Test Type | Coverage | Frequency |
|-----------|----------|-----------|
| **Unit Tests** | Individual functions (RAG retrieval, LLM parsing) | Every commit (CI/CD) |
| **Integration Tests** | End-to-end agent workflows | Daily |
| **Regression Tests** | Acceptance rate on fixed test set | Weekly |
| **Performance Tests** | Throughput (breaks/min), latency | Weekly |
| **Security Tests** | Prompt injection, data leakage | Monthly |
| **UAT** | Business stakeholder validation | Pre-pilot, pre-prod |

**Test Data**
- Synthetic breaks: 1,000 hand-crafted test cases covering edge cases
- Historical breaks: 10,000 randomly sampled from 2024-2025
- Adversarial prompts: 100 malicious inputs (SQL injection, prompt injection)

---

## 6. Monitoring & Observability

### 6.1 Model Performance Monitoring

**Real-Time Metrics (Prometheus)**
```python
from prometheus_client import Counter, Histogram, Gauge

# Counters
llm_requests_total = Counter('llm_requests_total', 'Total LLM API calls')
llm_errors_total = Counter('llm_errors_total', 'Total LLM errors')
analyst_decisions = Counter('analyst_decisions', 'Analyst decisions', ['decision_type'])

# Histograms
llm_latency = Histogram('llm_latency_seconds', 'LLM response time')
llm_tokens_used = Histogram('llm_tokens_used', 'Tokens per request', ['type'])

# Gauges
acceptance_rate = Gauge('acceptance_rate_7d', '7-day rolling acceptance rate')
avg_confidence = Gauge('avg_llm_confidence_7d', '7-day avg confidence')
```

**Dashboard Alerts**
```yaml
Alerts:
  - name: low_acceptance_rate
    condition: acceptance_rate_7d < 0.70
    severity: warning
    action: Email to ML team + Governance

  - name: high_error_rate
    condition: rate(llm_errors_total[5m]) > 0.05
    severity: critical
    action: PagerDuty alert + Email

  - name: token_budget_exceeded
    condition: sum(llm_tokens_used) > monthly_budget * 0.90
    severity: warning
    action: Email to project lead

  - name: latency_spike
    condition: llm_latency_seconds{quantile="0.95"} > 5
    severity: warning
    action: Investigate Azure OpenAI service health
```

### 6.2 Drift Detection

**Statistical Monitoring**
```python
import scipy.stats as stats

def detect_input_drift(current_month, baseline_month):
    """Kolmogorov-Smirnov test for input distribution drift."""
    
    # Numeric features
    for col in ['BREAK AMOUNT GBP', 'Age Days', 'ABS GBP']:
        ks_stat, p_value = stats.ks_2samp(
            current_month[col],
            baseline_month[col]
        )
        
        if p_value < 0.01:
            alert(f"Drift detected in {col}: p={p_value:.4f}")
    
    # Categorical features (Chi-square)
    for col in ['Asset Class', 'Type of break', 'Entity']:
        contingency_table = pd.crosstab(
            current_month[col],
            baseline_month[col]
        )
        chi2, p_value, dof, expected = stats.chi2_contingency(contingency_table)
        
        if p_value < 0.01:
            alert(f"Drift detected in {col}: p={p_value:.4f}")

# Run monthly
detect_input_drift(breaks_march_2026, breaks_baseline_2025)
```

---

## 7. Failure Recovery & Business Continuity

### 7.1 Degradation Modes

| Failure Scenario | Detection | Mitigation | Recovery Time |
|------------------|-----------|------------|---------------|
| **Azure OpenAI outage** | Health check fails 3x | Fallback to manual workflow; queue breaks for later processing | 0 seconds (instant) |
| **Vector DB unavailable** | Query timeout >10s | Use cached top-N frequent patterns; reduce RAG context | <1 minute |
| **JIRA API down** | 503 errors | Store JIRA updates in local queue; retry with exponential backoff | Auto-recovery |
| **Streamlit UI crash** | Process monitoring | Auto-restart (k8s); session state preserved in Redis | <30 seconds |
| **Model drift detected** | Weekly statistical tests | Freeze LLM suggestions; escalate to model risk team | 1-2 weeks (retrain) |

### 7.2 Disaster Recovery

**Backup Strategy**
- **Database:** Daily full backup, 6-hour incremental, 30-day retention
- **Vector store:** Weekly full backup, stored in Azure Blob (immutable)
- **Code:** Git repository (Azure DevOps), tagged releases
- **Configuration:** Infrastructure-as-Code (Terraform), version controlled

**RTO/RPO**
- Recovery Time Objective (RTO): 4 hours
- Recovery Point Objective (RPO): 6 hours

---

## 8. Token Optimization Strategies

### 8.1 Prompt Caching

```python
from openai import AzureOpenAI

# System prompt is constant - cached automatically by Azure OpenAI
SYSTEM_PROMPT_CACHED = """...[taxonomy and instructions]..."""

# For subsequent calls with same system prompt, 50% discount on input tokens
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT_CACHED},  # Cached
        {"role": "user", "content": dynamic_user_prompt}  # Not cached
    ]
)
```

### 8.2 Prompt Compression

**Before (verbose):**
```
Break Details:
- Trade Reference: TRD12345678
- Asset Class: Equity
- Type of Break: Settlement Mismatch
- Break Amount (GBP): 15000.00
- Age Days: 5
- Comments: Trade settled on T+3 instead of T+2 due to SSI mismatch...
```

**After (compressed - 30% token reduction):**
```
TRD12345678|Equity|Settlement Mismatch|£15K|5d|SSI mismatch T+3 vs T+2...
```

### 8.3 Smart RAG Retrieval

```python
def smart_rag_retrieval(break_data, max_chunks=3):
    """Retrieve only most relevant chunks based on confidence."""
    
    # First-pass: retrieve top-10 candidates
    candidates = vector_store.similarity_search(break_data, k=10)
    
    # Metadata filtering (same Asset Class, same Entity)
    filtered = [c for c in candidates if 
                c['Asset Class'] == break_data['Asset Class'] and
                c['Entity'] == break_data['Entity']]
    
    # Re-rank by recency (prefer recent resolutions)
    filtered = sorted(filtered, key=lambda x: x['Date'], reverse=True)
    
    # Return only top-3 to minimize context
    return filtered[:max_chunks]
```

**Token savings: 600 tokens → 300 tokens (50% reduction)**

---

## 9. Future Enhancements

### Phase 2 (Post-Pilot)
1. **Multilingual Support**
   - Fine-tune embedding model on multilingual financial text
   - Handle breaks from APAC (Japanese, Mandarin) and EMEA (French, German) entities

2. **Auto-Journaling**
   - For approved systematic break categories only
   - Requires separate governance approval
   - Integration with GL posting system

3. **Predictive Analytics**
   - ML model to predict breaks likely to age >30 days
   - Proactive escalation workflow

4. **Voice Interface**
   - Streamlit + Azure Speech SDK
   - Analyst can dictate break resolution via voice

### Phase 3 (Advanced Agentic)
1. **Multi-Agent Collaboration**
   - Specialist agents per asset class (Equity, FX, Fixed Income)
   - Consensus mechanism for cross-asset breaks

2. **Reinforcement Learning**
   - Learn from analyst feedback to improve suggestion quality
   - RLHF (Reinforcement Learning from Human Feedback)

3. **Break Prevention**
   - Real-time trade monitoring pre-settlement
   - LLM flags potential breaks before they occur

---

## 10. Appendices

### A. API Specifications

**POST /api/v1/breaks/process**
```json
Request:
{
  "break_id": "TRD12345678",
  "fields": {
    "Asset Class": "Equity",
    "Type of break": "Settlement Mismatch",
    "BREAK AMOUNT GBP": 15000.00,
    "Comments": "...",
    "JIRA DESC": "..."
  }
}

Response:
{
  "break_id": "TRD12345678",
  "llm_suggestion": {
    "root_cause": "Settlement Instruction (SSI) Error",
    "action": "Update SSI in system and reprocess settlement",
    "confidence": 87.5,
    "supporting_break_ids": ["TRD11223344", "TRD22334455", "TRD33445566"]
  },
  "status": "pending_review",
  "created_at": "2026-04-15T10:30:00Z"
}
```

### B. Environment Variables

```bash
# Azure OpenAI
AZURE_OPENAI_ENDPOINT=https://omrc-openai-prod.openai.azure.com
AZURE_OPENAI_API_KEY=<from Key Vault>
AZURE_OPENAI_DEPLOYMENT_NAME=gpt-4o-mini
AZURE_EMBEDDING_DEPLOYMENT=text-embedding-3-small

# Database
POSTGRES_HOST=omrc-db-prod.postgres.database.azure.com
POSTGRES_DB=breaks_ai
POSTGRES_USER=<from Key Vault>
POSTGRES_PASSWORD=<from Key Vault>

# Vector Store
CHROMADB_HOST=10.0.1.50
CHROMADB_PORT=8000

# Redis
REDIS_HOST=omrc-redis-prod.redis.cache.windows.net
REDIS_PASSWORD=<from Key Vault>

# Application
LOG_LEVEL=INFO
TOKEN_BUDGET_MONTHLY=200000000  # 200M tokens
RATE_LIMIT_PER_USER=1000  # tokens/min
```

### C. Glossary

| Term | Definition |
|------|------------|
| **RAG** | Retrieval-Augmented Generation - LLM technique that grounds responses in retrieved documents |
| **HITL** | Human-in-the-Loop - mandatory human approval step |
| **SR 11-7** | Federal Reserve guidance on Model Risk Management |
| **OMRC** | Operational Market Risk Control - middle office function |
| **SSI** | Standing Settlement Instructions - payment routing details |
| **B/S Cert** | Balance Sheet Certification - audit sign-off field |

---

**Document Owner:** [Your Name], Lead AI/ML Engineer - OMRC  
**Last Updated:** March 13, 2026  
**Next Review:** April 13, 2026 (post-Phase 1 completion)