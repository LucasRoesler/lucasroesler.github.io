---
date: "2013-06-21T11:27:27-04:00"
title: "Resume"
description: "Long Form Resume for Lucas Roesler"
layout: "single"
---

### Lucas Roesler
Berlin, DE \
<roesler.lucas@gmail.com>\
<https://github.com/LucasRoesler>\
<https://www.linkedin.com/in/lucasroesler/>

---
#### Technical  Skills
* Languages: Go, Python, SQL, Javascript, Typescript, HTML, CSS
* Data: Snowflake, dbt, Apache Airflow, Data Vault 2.0, PostgreSQL, Apache Spark, Apache Iceberg, MongoDB
* AI: coding agent harnesses (Claude Code, Codex, opencode), the Claude Agent SDK, MCP, agent sandboxing, LLM evaluation, RAG
* Platform: Kubernetes, Docker, Terraform, AWS, Azure, GCP, Prometheus, Grafana, OpenTelemetry

---

#### Education
**Ph.D. in Mathematics**, *University of Connecticut (UConn)*, Storrs, Connecticut - **May 2012**

>Dissertation Title: "Algebras from surfaces and their derived equivalences"
>Advisor: Dr. Ralf Schiffler

**M.S. in Mathematics**, *University of Connecticut (UConn)*, Storrs, Connecticut - **May 2009**

**B.A. in Mathematics**, *Central Connecticut State University (CCSU)*, New Britain, Connecticut - **May 2007**

---

#### Experience
**Managing Partner and CTO**, Contiamo GmbH, Berlin Germany, - **2022 -- present**

> Contiamo converted from a venture-funded data product company into a data and AI consultancy.
> I am one of four managing partners and own the technical side of the firm, while staying hands-on
> in delivery. Engagements run end-to-end, from the business problem to rollout, staffed with small
> senior teams. Clients are large enterprises in telecommunications, real estate, energy,
> insurance, and consumer goods.
>
> **Firm leadership**
>
> - Set architecture and delivery standards across engagements, and act as the most active
>   technical reviewer across client codebases.
> - Own engineering and AI standards: the internal agent tooling marketplace, the core agent
>   skills every project starts from, and the observability conventions clients inherit.
> - Exploratory work on capability-based agent sandboxing, built on Linux Landlock, so engineers
>   can run coding agents against real systems (notes, chat, ticketing, source control, cloud)
>   without exposing cloud credentials, SSH keys, or other tools' secrets.
> - Authored the engineering progression framework the firm runs on: four growth dimensions
>   (scope, autonomy, complexity, time horizon) across four bands with sub-levels, which also
>   drives candidate leveling and anchors salary review. Presented it to the partners and the team.
> - Designed and led the engineering hiring process end to end, including the interview rounds.
> - Shaped and rolled out the firm's one-on-one framework: a biweekly cadence the engineer drives,
>   with a quarterly career conversation, kept separate from project status reporting.
> - Built the internal systems the partners run the business on: the capacity and pipeline
>   dashboard covering utilization, allocation, and weighted pipeline, and the AI tooling cost
>   dashboard that moved the firm onto managed accounts.
> - Argued the firm's technical direction in partner strategy sessions, including building on
>   existing agent harnesses over bespoke agent frameworks, and the guardrails (written SLA,
>   capacity caps, monitoring access) required before signing maintenance contracts.
>
> **Enterprise procurement data platform** (telecommunications, 2024 -- present)
>
> - Technical lead migrating procurement reporting off a vendor SaaS onto Snowflake and Power BI.
> - Designed the Data Vault 2.0 warehouse (raw vault, business vault, use-case marts) and the
>   repeatable, automated process that builds it: templated dbt models over the AutomateDV macros,
>   Airflow orchestration, Terraform-provisioned environments, ingestion through Azure blob
>   storage, and DataHub for catalog. Five environments follow the same path to production.
> - Now integrating AI natively into the data engineering process itself.
> - Authored the data strategy and the business justification for Data Vault over a
>   SAP-native approach.
> - Wrote the continuous delivery counter-proposal that replaced the operations vendor's
>   seven-gate definition of done and deployment handover model.
> - Cut pipeline cost and runtime: per-DAG and per-task Snowflake warehouse overrides plus
>   partitioned incremental loads brought daily jobs that overran a 05:00 report refresh back
>   inside the window.
> - Chose Snowflake over Databricks in a proof of concept weighed against the client's
>   security requirements.
> - Reviewed a production data incident (silent mutation of effective dates producing 177
>   duplicate keys across five satellites) and specified the fix. Set strict schema and format
>   validation at ingest, so a bad file stops its source rather than corrupting the vault.
> - Designed a masked, cleaned production clone shared into the development account so engineers
>   could model against realistic data without production access, and pushed back on
>   over-broad production grants.
>
> **Data store replatform** (real estate advisory, 2023 -- 2024)
>
> - Designed and built the replacement for a legacy data store: AWS DMS change data capture from
>   MSSQL into Snowflake, a dbt Data Vault, and Airflow on AWS MWAA with astronomer-cosmos,
>   all provisioned in Terraform.
> - Solved foreign key metadata for a source system without referential constraints, and built a
>   Terraform generator for per-pipeline resources.
>
> **Property validation service** (real estate advisory, 2025 -- present)
>
> - Engineering lead on a geocoding and address validation service combining Photon for
>   normalization and Nominatim for polygons behind one API, with a compliant commercial fallback.
> - Benchmarked against an annotated address set and load tested to 94,000 addresses, which
>   surfaced an instability in the search cluster. Role-based access control with directory sync.
>
> **AI architecture assessment** (medtech, 2026)
>
> - Assessment for a European clear-aligner group whose patient-facing voice and chat agents had
>   outgrown their original design: roughly ten services stitched together, no monitoring, and
>   unrestricted production deploys.
> - Delivered a current state review, a target architecture, and the migration path I argued for,
>   since the client cannot stop the running system to switch over.
> - Named the lock-in each recommendation creates and the exit path from it, rather than avoiding
>   hosted components a resource-limited team would otherwise have to operate.
>
> **Marketing personalization warehouse** (telecommunications, 2023 -- 2024)
>
> - Kafka into Spark Structured Streaming into Apache Iceberg on S3, modeled with dbt over Athena,
>   deployed with Helm and ArgoCD.
> - A production ranking job covering more than 22 million user profiles in about two hours.
> - Cut Spark cost, removed duplicate events after a crash loop, and resolved a broadcast join
>   that exceeded the 8GB limit. Authored and delivered the team knowledge transfer.
>
> **Applied AI delivery**
>
> - Design reviews and architecture for a procurement knowledge graph agent and a purchase order
>   copilot, including a graph storage review and a semantic views assessment.
> - Argued the firm's position against hand-built agent frameworks and for building on existing
>   harnesses such as the Claude Code SDK.
> - Internal AI tooling: a shared plugin and skill marketplace, automated multi-model code review
>   with Claude, Gemini, and Codex as house practice, and a capacity planning application.
>
> *Snowflake - dbt - Airflow - Data Vault 2.0 - Terraform - Python - Go - Azure - AWS - Spark - Iceberg*

**Software Architect and Team Lead**, Contiamo GmbH, Berlin Germany, - **2017 -- 2022**

> - Owned a machine-learning-as-a-platform product: a cloud native platform on which data
>   engineers built, deployed, and supported machine learning models.
> - It integrated with the other half of the product, a SQL gateway that spoke the Postgres wire
>   protocol, translated queries, and supported cross-database joins, together with the
>   associated data catalog.
> - Designed and managed authentication and authorization for the data catalog and the SQL proxy.
> - Team lead for the API, Authentication, Application, and Platform team.
> - Responsible for running daily stand-ups and one-on-ones.
> - Contribute to the development of the product roadmap to prioritize features/projects.
> - Create and review architectural decisions and implementation plans.
> - Design and build the APIs and data model for the Data Catalog.
> - Developed a release process to enable self-hosted on-premise installations of Contiamo.
> - Lead the adoption of Kubernetes and Helm as core components of this release process.
> - Lead the adoption of OpenTracing and Prometheus to ensure the application is observable out-of-the-box.
>
> *Go - Postgres - Kubernetes*

**Core Developer**, OpenFaaS, - **2017 -- 2023**

> - Implemented the Secrets, Logs, and Profiles features.
> - General contributions, bug fixes, and code reviews.
> - Significant contributions to the `faas-cli`, `faas-netes`, `faasd`, `faas-provider`, the `certifier`, and the Python function SDK components.
>
> *Go - Kubernetes - Docker - Containerd*

**Director of Engineering**, Teem (formerly EventBoard, acquired by WeWork), Salt Lake City, Utah - **2016 -- 2017**

> - Manage the workflow of the engineering team and work with stakeholders to establish project priorities and timelines.
> - Work with Support to troubleshoot and resolve or assign issues.
> - Support and drive development and adoption of architectural, coding, and security standards/governance.
> - Contribute to the development of the product roadmap to prioritize features/projects.
> - Manage and grow our team of developers.
> - Build and maintain dev tooling.
> - Backend development.
>
> *Python - Django - PostgresSQL - AWS*

**Lead Developer**, EventBoard, Salt Lake City, Utah - **2015 -- 2016**

> - Django web development.
> - Designed and built user management and permissions systems.
> - Designed API documentation system using Swagger.
> - Increase database migration stability.
> - Manage development teams.
>
> *Python - Django - PostgresSQL - AWS*

**Lead Developer**, JobDash, Salt Lake City, Utah - **2013 -- 2015**

> - AngularJS and Django web development.
> - Designed and built an analytics system in PostgreSQL and MongoDB.
> - Manage and implement development processes for a team of developers.
> - Manage server uptime and deployment on AWS and DigitalOcean with Fabric.
> - Decrease page load times by a factor of 10.
>
> *Python - Django - AngularJS - PostgreSQL - MongoDB - AWS - LaTeX*

**Software developer**, SAIC, Tucson, Arizona - **2013**

> - Researched and implemented new feature detection algorithms in Python and C++.
> - Managed the transition of projects from CVS and SVN to Git.
> - Implemented hooks to enforce guidelines and workflow procedure.
> - Implemented new features for active websites in Django.
>
> *Python - Django - C++ - Git*

**Web developer**, *Pedagogy in Large Lectures*, UConn, Storrs, Connecticut - **2011 -- 2012**

> - Designed, created, and managed the webpage, videos, surveys, and raw data reports
>   for the Pedagogy in Large Lectures research project.
> - Managed the transition to new developers.
>
> *PHP - MySQL - Javascript - LaTeX*

**Web developer**, *Mathematics Department*, UConn, Storrs, Connecticut - **2009 -- 2012**

> - Created secure forms for administrative use.
> - Updated and created new secure forms for student use.
> - Implemented basic security for existing web forms.
> - Created extensive user and developer documentation.
>
>  *PHP - MySQL - Javascript*

<!-- #### Public Projects

**Django Encrypted Json Field** - https://github.com/LucasRoesler/django-encrypted-json

> - Store JSON in Postgres with encrypted values and plain text keys.
> - Allows partial encryption, you can specify a list of keys to skip.

**Django Cryptographic Fields** - https://github.com/foundertherapy/django-cryptographic-fields

> - Contributions to allow key rotation and general improvements. -->
---
#### Open Source

- [**crit**](https://github.com/LucasRoesler/crit) - review loop between a developer and a coding agent, in Go.
  Inline comments on code, plans, and running pages feed structured human feedback back to the agent.
- [**psst**](https://github.com/LucasRoesler/psst) - secrets manager that injects credentials into agent commands without exposing the values.
- [**openfaas-loki**](https://github.com/LucasRoesler/openfaas-loki) - Loki log provider for OpenFaaS.

---
#### Talks / Publications
**Event Driven Postgres with OpenFaas**, Livestream with Alex Elis - **2021-02-02**
> We discuss how you can combine the Postgres WAL, NATS, and OpenFaaS to create a lightweight event driven architecture. Including a short demo of a simple "event echo" to prove the concept. https://www.youtube.com/watch?v=nGOZLOjg5Zs

**OpenFaaS Blog**
> - [*Time travelling through your OpenFaaS Function logs*](https://www.openfaas.com/blog/openfaas-loki-into/) - **2020-06-06**
> - [*How to Serve Data Science Models with OpenFaaS*](https://www.openfaas.com/blog/python-datascience-with-openfaas/) - **2020-06-04**
> - [*How to split large Python Functions across multiple files*](https://www.openfaas.com/blog/multifile-python-functions/) - **2019-01-29**

**Merging Quickly in a Cloud Native World**, Kubecon, Barcelona - **2019**
> You are really excited by a new project, you have a great idea to contribute, you click submit ... and wait. The latest StackOverflow community report states that 80% of surveyed developers are coding in their free time but only 44% are contributing to open source projects. While this number seems good, can we do better? How can we ensure that both contributors and maintainers enjoy and benefit from participating?
>
> In this presentation Lucas goes behind the scenes of the OpenFaaS project to share his knowledge from contributing to OpenFaaS and becoming a core contributor. We will walk through the standard git flow to understand what the maintainers of a large open-source project are looking for in new contributions, what we are doing to improve the experience, what the Kubernetes community at large is working on to help, and what you can do to get your pull request merged more quickly.
> ([Summary](https://kccnceu19.sched.com/event/MPZM/merging-quickly-in-a-cloud-native-world-lucas-roesler-contiamo)), ([Recording](https://youtu.be/d0E7PSlkBbs)) and ([Slides](https://static.sched.com/hosted_files/kccnceu19/e2/merging-quickly.pdf))

---
#### Other Experience

**Visiting Instructor in Mathematics**, *Lebanon Valley College*, Annville, Pennsylvania - **2012 -- 2013**

> *Finite Mathematics - Calculus 1 - Differential Equations*

**Teaching Assistant**, *UConn*, Storrs, Connecticut - **2007 -- 2012**

>*Discrete Mathematics - Mathematical Modeling - Calculus 1a - Calculus 1 - Calculus 2 - Business Calculus*

**Research Assistant**, *Pedagogy in Large Lectures*, UConn, Storrs, Connecticut - **2011 -- 2012**

>Integrated online videos and flashcards to improve learning in large lectures. Created content for, implemented, and maintained the web resources for displaying content and collecting survey data.

**Implementation of Crystals of Generalized Young Walls**, a method to construct combinatorial mathematical objects called crystals in the open source computer algebras system [Sage].

> Source available at <https://bitbucket.org/Theaxer/sage>
> Accepted for inclusion into the Sage combinat branch: <http://trac.sagemath.org/sage_trac/ticket/14130>
>
> *Python - Sage*


#### Other Publications

1. L\. David-Roesler. *The AG-invariant for m-angulations*, submitted, available online: [arXiv:1210.6087]

2. L\. David-Roesler. *Derived Equivalence in Surface Algebras via Graded Equivalence*, 30 pages, to appear in Algebr. Represent. Theory. [doi:10.1007/s10468-012-9384-9][].

3. L\. David-Roesler and R. Schiffler. *Algebras from surfaces without punctures*. J. Algebra 350 (2012). pp. 218-244. [doi:10.1016/j.jalgebra.2011.10.034].

---

#### Interests
* Rock climbing/bouldering, home brewing beer, music, hiking, reading, tabletop gaming, and participating in open source development.

  [Sage]: http://www.sagemath.org/ "Sage"
  [Linkedin]: http://www.linkedin.com/in/lucasroesler/ "Linkedin"
  [BitBucket]: https://bitbucket.org/LDavRoe "BitBucket"
  [doi:10.1016/j.jalgebra.2011.10.034]: http://www.sciencedirect.com/science/article/pii/S0021869311006211 "doi:10.1016/j.jalgebra.2011.10.034"
  [doi:10.1007/s10468-012-9384-9]: http://link.springer.com/article/10.1007%2Fs10468-012-9384-9 "doi:10.1007/s10468-012-9384-9"
  [arXiv:1210.6087]: http://arxiv.org/abs/1210.6087 "arXiv:1210.6087"
  [arxiv:results]: https://arxiv.org/search/?query=David-Roesler&searchtype=author
