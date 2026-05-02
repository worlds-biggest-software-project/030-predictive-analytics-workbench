# Predictive Analytics Workbench — Feature & Functionality Survey

> Candidate #30 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| DataRobot | Commercial Enterprise | Proprietary; from ~$150,000/year | https://www.datarobot.com |
| H2O.ai (H2O-3 + Driverless AI) | Open Source + Commercial | H2O-3: Apache 2.0; Driverless AI: Proprietary (from ~$12,000/year) | https://h2o.ai |
| Alteryx | Commercial SaaS + Desktop | Proprietary; Designer Cloud from ~$4,950/user/year | https://www.alteryx.com |
| Pecan AI | Commercial SaaS | Proprietary; quote-based (mid-market) | https://www.pecan.ai |
| Akkio | Commercial SaaS | Proprietary; from $49/mo (Starter) | https://www.akkio.com |
| SageMaker Canvas | Commercial Cloud | Proprietary (AWS); pay-as-you-go | https://aws.amazon.com/sagemaker/ai/canvas/ |
| MindsDB | Open Source + Cloud | GPL-3.0 (core OSS); Proprietary (MindsDB Cloud) | https://mindsdb.com |
| AutoGluon (AWS) | Open Source | Apache 2.0 | https://auto.gluon.ai |

## Feature Analysis by Solution

### DataRobot

**Core features**
- Enterprise AutoML engine that evaluates hundreds of algorithms and thousands of hyperparameter combinations to produce a model leaderboard
- Automated data quality assessment, feature engineering, and feature selection with data enrichment from external sources
- Comprehensive MLOps: model deployment, A/B testing, champion-challenger comparisons, automated retraining on drift detection
- Model explainability: SHAP values, feature impact analysis, prediction explanations, and what-if scenario modelling
- Compliance documentation generator producing audit-ready reports on model development, data provenance, and performance metrics
- Agentic AI Suite (2025–2026 repositioning): enables frontline teams to build, deploy, and govern AI agents integrated with business processes
- Automated data drift detection, prediction drift analysis, and bias monitoring in production
- Full lifecycle coverage: data prep → model training → deployment → monitoring → retraining

**Differentiating features**
- Most comprehensive enterprise MLOps governance layer in the AutoML market, including bias monitoring and audit-ready compliance documentation
- Repositioning as a full-stack "AI agent platform" (2025) distinguishes it from pure AutoML vendors
- Champion-challenger model comparison in production is a feature few competitors match at this depth
- Compliance documentation generator directly addresses regulated-industry requirements (finance, pharma, insurance)

**UX patterns**
- Guided wizard-style workflow for model building targeting business analysts, not only data scientists
- Leaderboard UI shows model accuracy, explainability scores, and deployment readiness in a single view
- Insight explanations surface top features and prediction drivers for each model output
- Enterprise deployment: typical onboarding involves significant professional services; not self-service at SMB price points

**Integration points**
- Data connectors: major cloud warehouses (Snowflake, BigQuery, Redshift, Databricks), S3, SFTP, JDBC
- MLflow-compatible model registry for experiment tracking
- REST API for model scoring and deployment
- Salesforce, SAP, and other enterprise system integrations for prediction consumption
- PMML and ONNX model export for portability across scoring environments
- SSO, RBAC, and audit logging for enterprise security

**Known gaps**
- Price ($150,000+/year) excludes all but large enterprise buyers
- Pricing opacity requires sales engagement; no self-serve or freemium path
- AutoML transparency is limited for highly technical users who want to inspect model internals
- Requires dedicated MLOps teams to realise full platform value
- Time-series forecasting capabilities, while present, are not as specialised as dedicated forecasting tools

**Licence / IP notes**
- Fully proprietary. DataRobot holds patents in AutoML pipeline optimisation and automated feature engineering. Competing products should develop independent model selection, feature engineering, and explainability pipelines to avoid IP risk. No open-source components exposed.

---

### H2O.ai (H2O-3 AutoML + H2O Driverless AI)

**Core features**
- H2O-3 (OSS): distributed in-memory ML with linear scalability; AutoML module evaluates all algorithms and hyperparameters and produces a model leaderboard via R, Python, or web GUI
- H2O Driverless AI: fully automated feature engineering (detects feature interactions via evolutionary competition), model development, validation, and documentation
- Built-in best-practices guardrails preventing overfitting and common modelling pitfalls
- Machine learning interpretability (MLI): SHAP values, partial dependence plots, surrogate models, and reason codes for every prediction
- Automatic ML pipeline documentation for compliance and audit
- H2O.ai Visionary in 2025 Gartner Magic Quadrant for Cloud AI Developer Services
- H2O Wave: low-code Python framework for deploying interactive ML applications
- Multimodal foundation models for document AI announced 2024

**Differentiating features**
- H2O-3 AutoML is among the benchmark leaders for tabular prediction without any paid licence — accessible to any team
- Driverless AI's evolutionary feature engineering approach (genetic algorithm-based feature competition) is distinctively different from gradient-based feature selection used by most peers
- MLI (Machine Learning Interpretability) module is one of the most comprehensive explanation toolkits among AutoML vendors
- Open-source H2O-3 serves as a capable entry point; Driverless AI provides enterprise upgrade path

**UX patterns**
- H2O-3: available via Python/R API and Flow web GUI; technical users need Python/R proficiency
- Driverless AI: more automated but still requires domain expertise to configure experiment settings
- MLI explanations are visualised in interactive charts accessible to non-technical stakeholders
- Not a no-code product at the same accessibility level as Pecan AI or Akkio

**Integration points**
- Python and R APIs; REST scoring endpoint for deployed models
- H2O MOJO (Model Object Optimised): portable scoring artifact deployable in Java, Python, R, or Spark without H2O runtime dependency
- MLflow integration for experiment tracking and model registry
- HDFS, S3, and JDBC data source connectivity
- Spark integration for distributed feature engineering
- PMML export for scoring portability

**Known gaps**
- H2O-3 AutoML requires Python or R proficiency; no meaningful no-code UI for business analysts
- Driverless AI requires data science expertise to configure meaningfully; not self-service for non-technical users
- GUI (Flow for H2O-3) is functional but dated compared to modern SaaS tools
- Limited native time-series forecasting specialisation compared to AutoGluon TimeSeries or SageMaker Canvas
- Community support quality for H2O-3 varies; enterprise support gated behind paid tier

**Licence / IP notes**
- H2O-3: Apache 2.0 (permissive, safe for commercial use, patent grant included). H2O Driverless AI: fully proprietary with an annual licence fee. H2O MOJO scoring artifacts can be distributed independently of the H2O runtime under the Apache licence. H2O.ai holds patents in automated feature engineering; Driverless AI's evolutionary feature engineering method in particular should be examined for IP risk if building a similar evolutionary optimisation approach.

---

### Alteryx

**Core features**
- 300+ pre-built drag-and-drop workflow tools covering data preparation, blending, spatial analytics, predictive analytics, and ML
- No-code visual workflow builder (Designer) allowing business analysts to build end-to-end analytics pipelines without SQL or Python
- Alteryx Machine Learning: no-code cloud AutoML with automated feature engineering using Deep Feature Synthesis
- Education Mode in AutoML: explains concepts (overfitting, feature importance) while users solve problems — unique teaching-while-doing UX
- Alteryx One platform (2026): unified environment combining Designer, Server (deployment), Connect (cataloging), and Promote (model management)
- In-warehouse execution for BigQuery (2025 addition): runs data prep logic directly inside the warehouse without data movement
- Spatial analytics: geocoding, trade area analysis, drive-time isochrones

**Differentiating features**
- End-to-end data prep + ML in a single no-code workflow product — no switching between separate ETL and ML tools
- Education Mode that teaches ML concepts contextually as analysts build models (unique in the market)
- Deep Feature Synthesis (DFS) automated feature engineering that builds features from relational data across multiple tables
- Spatial analytics depth is unmatched among SMB/mid-market analytics tools
- In-warehouse execution reduces data movement cost and latency for cloud warehouse users

**UX patterns**
- Canvas-based drag-and-drop workflow design; tool palette on the left, workflow canvas in the center
- Strongly favoured by citizen data scientists and operations analysts in finance, healthcare, and retail
- Results and model diagnostics presented inline within the workflow canvas
- Not designed for real-time prediction serving; outputs are batch-oriented

**Integration points**
- Data connectors: 300+ including Snowflake, BigQuery, Redshift, Databricks, Oracle, SAP, Salesforce
- In-warehouse execution: BigQuery (launched 2025); Snowflake support announced
- Server for workflow scheduling and API-based prediction serving
- Python and R tool nodes for custom code within workflows
- Model output connectors: Salesforce, Tableau, Power BI

**Known gaps**
- Per-user pricing ($4,950+/user/year) is expensive for large analyst teams
- Not designed for real-time prediction endpoints; batch-oriented architecture limits operational AI use cases
- AutoML capabilities are less powerful than DataRobot or H2O Driverless AI for complex modelling tasks
- Acquired by Clearlake Capital and Insight Partners (2024); ongoing restructuring creates product roadmap uncertainty
- Limited deep learning or foundation model support

**Licence / IP notes**
- Fully proprietary SaaS and desktop product. Deep Feature Synthesis (DFS) is an academic method originating from MIT CSAIL (Kanter and Veeramachaneni, 2015) and published in open literature; implementations are widespread and the method itself is not proprietary. No specific Alteryx patent encumbrances identified that would affect an independent implementation. Standard enterprise SaaS terms apply.

---

### Pecan AI

**Core features**
- No-code predictive analytics platform: users upload data or connect a cloud warehouse and specify what to forecast
- LLM-powered co-pilot (Predictive GenAI) that understands business questions, auto-builds predictive models, defines KPIs, and prepares data conversationally
- Automated end-to-end pipeline: data ingestion → feature engineering → model training → deployment → production prediction serving
- Supported use cases: churn prediction, LTV modelling, lead scoring, demand forecasting, and propensity scoring
- Connects directly to cloud data warehouses (Snowflake, BigQuery, Redshift) and CSV/Excel uploads
- Production-ready model deployment without requiring a data engineering team

**Differentiating features**
- Conversational AI co-pilot that translates a business question (e.g., "which customers are likely to churn in the next 30 days?") directly into a deployed model — lowest-friction path from question to prediction in the market
- Full production deployment (scoring at warehouse scale) is included in the no-code workflow, not an afterthought
- Designed specifically for business analysts and marketing/ops teams without ML knowledge

**UX patterns**
- Conversational onboarding: co-pilot asks clarifying questions before building the model
- Model output delivered as scored predictions written back to the connected warehouse, not as a downloadable file
- Business-oriented result presentation (probability scores, segment distributions, top contributing features)
- Limited interactive model inspection for technical users who want to examine internals

**Integration points**
- Snowflake, BigQuery, Redshift (direct connector); CSV/Excel upload
- Prediction output written back to connected warehouse for downstream BI tool consumption
- REST API for prediction serving in operational applications
- Salesforce and CRM integration for lead scoring use cases

**Known gaps**
- Model transparency and explainability are limited; Pecan prioritises ease of use over technical interpretability
- Not suitable for complex multi-step or custom forecasting scenarios requiring bespoke feature engineering
- Time-series specialisation is present but less sophisticated than dedicated forecasting tools
- Pricing is quote-based with no transparent tiers, creating friction for SMB evaluation
- Limited support for unstructured data, text, or multimodal inputs

**Licence / IP notes**
- Fully proprietary SaaS. No open-source components. No patent encumbrances identified in public sources.

---

### Akkio

**Core features**
- No-code AI platform for non-technical marketing agency and business teams
- Supports forecasting, lead scoring, churn prediction, and customer segmentation via drag-and-drop interface
- Model training in as little as 10 seconds; no per-training cost (pay for predictions, not training runs)
- Deployment integrations: Salesforce, Google Sheets, Snowflake, and REST API
- Chat Explore: natural language interface for exploratory data analysis before model building
- Lowest price point in the no-code ML market ($49/mo Starter)

**Differentiating features**
- Fastest model training time in the no-code market (10 seconds for many datasets)
- No-cost-per-training model encourages experimentation without financial penalty
- Chat Explore provides a natural language EDA layer before committing to a model
- Price point ($49–$99/mo) makes it accessible to individual analysts and micro-teams

**UX patterns**
- Minimal UI emphasising speed: upload data, select target column, click train
- Workflow-style deployment to Salesforce and Google Sheets for non-technical operationalisation
- Not designed for complex data preparation or multi-table joins
- No education or explanation layer beyond basic feature importance

**Integration points**
- Data input: CSV upload, Google Sheets, Snowflake
- Model deployment: Salesforce, Google Sheets, REST API
- Zapier integration for lightweight automation

**Known gaps**
- Very limited for complex multi-step forecasting or multi-table feature engineering
- No time-series-specific forecasting workflows; general classification and regression focus
- Model explainability depth is minimal — not suitable for regulated industries
- Limited data preparation tools; expects clean data input
- Enterprise security (SSO, audit logs, RBAC) only available on Enterprise plan
- Limited scalability for large datasets or high-volume prediction serving

**Licence / IP notes**
- Fully proprietary SaaS. No open-source components. No patent encumbrances identified.

---

### Amazon SageMaker Canvas

**Core features**
- No-code ML model builder within AWS SageMaker supporting regression, classification, time-series forecasting, NLP, and computer vision
- 300+ pre-built PySpark data transformation templates for data preparation without code
- Data Quality and Insight report: one-click anomaly detection (outliers, class imbalance, data leakage)
- Time-series forecasting with Chronos foundation models (Amazon's pre-trained time-series models via SageMaker JumpStart)
- Amazon Q Developer integration (2025): conversational guidance throughout the ML workflow from data prep to deployment
- Model sharing: share predictions with Amazon QuickSight to combine BI and predictive data in dashboards
- Pay-as-you-go billing (Canvas sessions billed per hour + training compute); no upfront licence cost

**Differentiating features**
- Chronos foundation models enable zero-shot time-series forecasting — no historical training data required for the model itself (user data is used for fine-tuning context)
- Amazon Q Developer conversational assistant embedded in the ML workflow is unique among no-code tools
- Native QuickSight integration merges predictions with BI dashboards without data engineering
- AWS ecosystem integration (S3, Athena, Redshift, Glue) provides seamless access to enterprise data already in AWS

**UX patterns**
- Guided point-and-click workflow; each step (import → build → analyse → predict) presented in sequence
- Data Quality report surfaced before model building, not after, encouraging clean-data-first habits
- Model leaderboard (similar to AutoML-style) shows candidate models ranked by accuracy metrics
- Designed for business analysts; no Python or SQL required for standard workflows

**Integration points**
- AWS native: S3, Athena, Redshift, RDS, SageMaker Feature Store, SageMaker Pipelines
- SageMaker endpoints for real-time inference after Canvas model deployment
- QuickSight for BI dashboard integration with prediction outputs
- SageMaker JumpStart for Chronos and other foundation model access
- Wrangler for advanced data preparation with code if needed

**Known gaps**
- Locked to AWS ecosystem; no multi-cloud or on-premises deployment
- Limited customisation for advanced users who need bespoke architectures
- Pay-as-you-go pricing can become expensive for frequent large training runs if not carefully managed
- Chronos zero-shot accuracy varies significantly by domain; fine-tuning requires more technical knowledge
- Not suitable for organisations not already invested in the AWS data stack
- Model portability outside AWS is limited despite ONNX support claims

**Licence / IP notes**
- Fully proprietary AWS service. Chronos (the time-series foundation model) is available on GitHub under the Apache 2.0 licence (amazon-science/chronos-forecasting) for independent use outside SageMaker. ONNX export supported for trained models. No patent encumbrances identified in public sources for building competing no-code ML tools; Chronos weights are open under Apache 2.0.

---

### MindsDB

**Core features**
- SQL-native ML interface: users create, query, and manage predictive models using standard SQL syntax directly against connected databases
- "AI Data Vault": universal SQL-like interface abstracting 200+ data sources for AI agents to query securely
- MCP (Model Context Protocol) server support (re-architected Q1 2025): turns MindsDB into a universal adapter for AI agents (Claude, GPT-4, etc.) to query any backend data source
- LLM and RAG (retrieval-augmented generation) support integrated into the SQL query layer
- Knowledge Bases for vector search within the SQL interface (major improvements July 2025)
- GUI evolved into a full IDE for AI: tabbed interface with drag-and-drop organisation, session persistence, and per-tab storage
- Full redesign in 2025 from prediction-focused SQL tool to a universal AI data hub platform

**Differentiating features**
- Unique SQL-native ML interface — the only tool where predictions are a SQL query (`SELECT forecast FROM model WHERE ...`), lowering the ML skill floor for SQL-proficient analysts
- MCP server positioning as a universal AI agent data adapter is differentiated from any other tool in this category
- SQL abstraction over 200+ data sources means ML models can be queried in the same SQL dialect as source data
- Pivot from AutoML to AI data orchestration platform in 2025 is architecturally distinct from all other tools reviewed

**UX patterns**
- SQL-first: primary users are data engineers and analysts comfortable with SQL, not a no-code audience
- Web IDE for composing and running SQL-based AI workflows with a modern tab interface
- GitHub integration for versioned AI workflow management
- Less suitable for business analysts who want a GUI-first no-code experience

**Integration points**
- 200+ data source connectors queryable via SQL interface
- MCP Server for AI agent integration (Claude Desktop, GPT-4-based agents, and others)
- LLM integrations: OpenAI, Anthropic, local models via Ollama
- REST API and Python SDK
- Available as OSS (self-hosted Docker) or MindsDB Cloud (managed SaaS)

**Known gaps**
- GPL-3.0 core licence is a significant concern for commercial use (see Licence section below)
- No-code UX for non-SQL users is minimal; requires SQL proficiency
- Time-series forecasting capabilities are present but not a primary focus after the 2025 platform pivot
- Limited model explainability (SHAP, feature importance) compared to DataRobot or H2O
- Model governance, drift monitoring, and production MLOps features are underdeveloped
- Platform identity shift in 2025 (from AutoML to AI data hub) creates uncertainty about future ML-specific feature investment

**Licence / IP notes**
- The MindsDB OSS core is licenced under GPL-3.0, which is strongly copyleft: any distribution of software that incorporates or links against GPL-3.0 code must release the entire combined work under GPL-3.0. This is a material commercial concern — building a proprietary product that ships MindsDB OSS is not legally permissible without a commercial licence from MindsDB. Teams building competing or complementary tools should avoid incorporating GPL-3.0 MindsDB code; instead, use the MCP server interface or REST API as an integration boundary. Some components may be under different licences (Apache 2.0 for specific modules); verify per-component licensing before use.

---

### AutoGluon (AWS)

**Core features**
- State-of-the-art open-source AutoML supporting tabular, text, image, time series, and multimodal data
- AutoGluon-Tabular: automated ensemble stacking of LightGBM, CatBoost, XGBoost, and neural nets via bagging, boosting, and multi-layer stacking
- AutoGluon-TimeSeries: combines statistical methods (ETS, ARIMA), tree-based models (LightGBM), deep learning (DeepAR, Temporal Fusion Transformer), and Chronos pretrained zero-shot models in an ensemble forecaster
- Three lines of code to train and serve a model — minimal API surface
- Research-backed: NeurIPS Spotlight 2025 (TabArena tabular benchmark), AutoML Conf 2025 (multi-layer stack ensembles for time series), Chronos-2 paper (Arxiv 2025)
- ONNX and TorchScript model export for production deployment portability
- Feature importance plots via Explain Tabular module

**Differentiating features**
- Chronos integration in TimeSeries provides zero-shot forecasting from a pre-trained foundation model — unique among open-source AutoML libraries
- NeurIPS Spotlight TabArena benchmark win (2025) provides peer-reviewed evidence of best-in-class tabular accuracy
- Multi-layer stack ensembling for time series (AutoML Conf 2025) delivers production-grade forecast accuracy at zero licence cost
- Apache 2.0 licence makes it fully safe for commercial use with patent grant included

**UX patterns**
- Python-only API; no GUI or no-code interface
- Designed for data scientists comfortable with Python; not accessible to business analysts
- Minimal configuration required for strong default results ("fit and forget" philosophy)
- Outputs are Python model objects; operationalisation requires separate deployment infrastructure

**Integration points**
- Python package (pip installable); runs locally, on-premises, or on any cloud
- AWS SageMaker integration for managed training and inference
- ONNX and TorchScript export for cross-framework deployment
- MLflow compatible for experiment tracking
- Tabular connectors: pandas DataFrames (any CSV, Parquet, or database-sourced data)

**Known gaps**
- No GUI or no-code interface; exclusively a Python library
- No built-in production serving infrastructure; model deployment requires separate tools (FastAPI, SageMaker endpoints, etc.)
- No model monitoring, drift detection, or automated retraining
- No native MLflow-based model registry (requires manual MLflow integration)
- Time-series module is powerful but requires Python expertise to tune correctly
- No enterprise support tier; community support via GitHub Issues only

**Licence / IP notes**
- Apache 2.0: permissive, patent grant included, fully safe for commercial use without open-sourcing derivatives. Chronos model weights (used within AutoGluon-TimeSeries) are also available under Apache 2.0 (amazon-science/chronos-forecasting on GitHub). No patent encumbrances identified. The strongest open-source AutoML licence posture in this survey.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Automated model training for at least tabular classification and regression use cases
- Feature importance / SHAP-based explanation for every model output
- Model accuracy leaderboard comparing candidate algorithms
- Train/test split management and cross-validation
- Batch prediction export to CSV or connected data warehouse
- Data quality check (null rates, outliers, class imbalance) surfaced before model training
- REST API for model scoring in downstream applications

### Differentiating Features
- No-code, conversational workflow guided by an LLM co-pilot (Pecan AI, SageMaker Canvas with Q)
- Automated time-series forecasting with pre-trained foundation model support (Chronos in AutoGluon, SageMaker Canvas)
- Education Mode teaching ML concepts contextually during model building (Alteryx)
- Deep Feature Synthesis across relational table joins for automated feature engineering (Alteryx, H2O Driverless AI)
- Production MLOps with drift detection, champion-challenger testing, and automated retraining (DataRobot)
- Compliance documentation generator for audit-ready model provenance reports (DataRobot)
- SQL-native ML interface enabling prediction queries without leaving the analyst's SQL workflow (MindsDB)
- MCP-based AI agent interoperability (MindsDB, SageMaker via JumpStart)

### Underserved Areas / Opportunities
- LLM-as-copilot for the entire modelling workflow: guiding non-technical users conversationally, explaining train/test split, flagging target leakage, and recommending model types for a described business question — without requiring ML knowledge
- Context-aware feature engineering from natural language domain description ("this is monthly SaaS subscription data; predict churn 90 days out") producing semantically meaningful features (rolling windows, lag features, cohort flags) that blind AutoML misses
- Plain-English interpretable forecasting narratives explaining *why* a forecast is what it is, making model outputs usable for stakeholders who distrust black-box numbers
- No-code time-series forecasting for operations teams (demand planning, inventory, revenue modelling) at SMB/mid-market price points — this use case is underserved between free Python libraries (AutoGluon, Prophet) and enterprise tools ($150K+/year DataRobot, SAS)
- Automated model drift management without MLOps expertise: drift detection, retraining trigger, and plain-English notification that a model has become unreliable — production ML for teams without dedicated MLOps engineers

### AI-Augmentation Candidates
- LLM assistant guiding users through the modelling workflow step-by-step in natural language (explaining concepts, flagging issues)
- Domain-context-driven feature engineering: using a natural language description of the business problem to generate domain-specific features the AutoML engine would not discover blindly
- Auto-generated plain-English prediction explanation narratives for each forecast output
- Automated drift detection with natural language summary of what changed and why the model is degrading
- Conversational EDA before model building (Akkio's Chat Explore direction, but deeper)
- LLM-powered code generation for custom model deployment scripts and integration with existing data pipelines

---

## Legal & IP Summary

Among the tools analysed, AutoGluon (Apache 2.0) and H2O-3 (Apache 2.0) are the safest open-source foundations for a commercial predictive analytics workbench; both licences are permissive and include explicit patent grants. MindsDB's GPL-3.0 core is a material concern: incorporating GPL-3.0 code in a distributed commercial product requires releasing the entire combined product under GPL-3.0 unless a commercial licence is obtained from MindsDB, Inc. DataRobot holds patents in AutoML pipeline optimisation and automated feature engineering; H2O.ai holds patents in evolutionary feature engineering (Driverless AI); any workbench that implements similar algorithmic approaches should conduct a freedom-to-operate analysis before commercialisation. Deep Feature Synthesis (Alteryx) originated as an academic method (Kanter & Veeramachaneni, MIT CSAIL, 2015) and is not proprietary to Alteryx; open implementations exist. Amazon Chronos model weights are Apache 2.0 and freely usable. PMML is a royalty-free DMG standard; ONNX is an Apache 2.0 Linux Foundation standard; MLflow is Apache 2.0. SHAP and LIME are both MIT-licenced Python libraries with no patent encumbrances. No FRAND-encumbered standards were identified in this space.

---

## Recommended Feature Scope

**Must-have (MVP)**:
- Automated model training for tabular classification, regression, and time-series forecasting from CSV, cloud warehouse, or API data input
- LLM-powered conversational assistant guiding users through the modelling workflow: explaining each step, flagging data quality issues (nulls, class imbalance, leakage), and recommending the right model type for a described business question
- Model accuracy leaderboard with auto-selected best model and plain-English explanation of why it was chosen
- SHAP-based feature importance and plain-English prediction explanation for every model output ("Revenue is projected to decline 12% because...")
- Batch prediction export to CSV or write-back to connected cloud warehouse
- Basic drift detection: notification when model accuracy degrades below a configurable threshold

**Should-have (v1.1)**:
- Domain-context-driven feature engineering: users describe their business problem in natural language, and the workbench generates semantically meaningful features (rolling windows, lag features, cohort flags, seasonality components) beyond what blind AutoML produces
- Automated retraining pipeline triggered by drift detection, requiring no MLOps configuration from the user
- Plain-English forecasting narratives auto-generated for each prediction run, suitable for stakeholder presentations and board reporting
- REST API endpoint for real-time prediction serving in downstream operational applications
- MLflow-compatible experiment tracking and model registry for data science teams

**Nice-to-have (backlog)**:
- No-code time-series-specific workflow for operations teams (demand planning, inventory forecasting, revenue modelling) with Chronos-based zero-shot bootstrapping for cold-start datasets
- Compliance documentation generator producing audit-ready model development reports (train/test provenance, feature selection rationale, performance metrics)
- Champion-challenger comparison in production: run two model versions in parallel with traffic splitting and auto-promote winner
- PMML and ONNX model export for portability to scoring environments outside the workbench
- Education Mode: contextual ML concept explanations embedded in the workflow for citizen data scientists building their first models
