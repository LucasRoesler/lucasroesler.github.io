---
date: "2013-06-21T11:27:27-04:00"
title: "Resume"
description: "Resume for Lucas Roesler"
slug : "resume"
type : "resume"
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

**M.S. in Mathematics**, *University of Connecticut (UConn)*, Storrs, Connecticut - **May 2009**

**B.A. in Mathematics**, *Central Connecticut State University (CCSU)*, New Britain, Connecticut - **May 2007**

---

#### Experience
**Managing Partner and CTO**, Contiamo GmbH, Berlin Germany, - **2022 -- present**

- One of four managing partners of a data and AI consultancy that delivers end-to-end
  engagements for large enterprises in telecommunications, real estate, energy, and insurance.
- Own engineering and AI standards across the firm: the internal agent tooling marketplace,
  the core agent skills every project starts from, and the observability conventions clients inherit.
- Exploratory work on capability-based agent sandboxing, so engineers can run coding agents
  against real systems without exposing cloud credentials and SSH keys.
- Authored the engineering progression framework the firm runs on: four growth dimensions across
  four bands, driving candidate leveling and salary review. Designed and led the hiring process
  end to end, and rolled out the firm's one-on-one framework.
- Built the internal systems the partners run the business on: the capacity and pipeline dashboard
  (utilization, allocation, weighted pipeline) and the AI tooling cost dashboard.
- Technical lead on a procurement data platform for a European telecommunications group,
  migrating reporting off a vendor SaaS onto Snowflake. Designed the Data Vault 2.0 warehouse
  and the repeatable, automated delivery process around it: templated dbt models, Airflow
  orchestration, and Terraform-provisioned environments.
- Replaced a seven-gate release handover with a continuous delivery model, and brought daily
  pipelines that overran the morning reporting window back inside it.
- Now integrating AI natively into the data engineering process itself.
- Architecture assessments for clients whose AI systems have outgrown their original design,
  including a European medtech group running patient-facing voice and chat agents: current state
  review, target architecture, and a migration path that keeps the running system online.
- Designed and replatformed a real estate advisory group's data store: change data capture from
  MSSQL into Snowflake, a dbt Data Vault, and Airflow on AWS MWAA.
- Built a marketing personalization warehouse on Kafka, Spark Structured Streaming, and Iceberg,
  with a production job that ranks over 22 million user profiles.
- *Snowflake - dbt - Airflow - Terraform - Python - Go - Azure - AWS*

**Software Architect and Team Lead**, Contiamo GmbH, Berlin Germany, - **2017 -- 2022**

- Owned a machine-learning-as-a-platform product: a cloud native platform for data engineers to
  build, deploy, and support machine learning models.
- It integrated with a SQL gateway that spoke the Postgres wire protocol, translated queries, and
  supported cross-database joins, together with the associated data catalog.
- Designed and managed authentication and authorization for the data catalog and the SQL proxy.
- Team lead for the API, Authentication, Application, and Platform team, running stand-ups and
  one-on-ones and contributing to the product roadmap.
- Created and reviewed architectural decisions and implementation plans.
- Developed a release process for self-hosted on-premise installations, and led the adoption of
  Kubernetes and Helm as its core components.
- Led the adoption of OpenTracing and Prometheus so the application was observable out of the box.
- *Go - Postgres - Kubernetes*

**Core Developer**, OpenFaaS, - **2017 -- 2023**

- Community driven open source serverless framework that runs on Kubernetes and Containerd
- Implemented the Secrets, Logs, and Profiles features.
- General contributions, bug fixes, and code reviews.
- *Go - Kubernetes - Docker - Containerd*

**Director of Engineering**, Teem (formerly EventBoard, acquired by WeWork), Salt Lake City, Utah - **2016 -- 2017**

- Manage the workflow of the engineering team and work with stakeholders to establish project priorities and timelines.
- Work with Support to troubleshoot and resolve or assign issues.
- Support and drive development and adoption of architectural, coding, and security standards/governance.
- Contribute to the development of the product roadmap to prioritize features/projects.
- Manage and grow our team of developers.
- Build and maintain dev tooling.
- Backend development.
- *Python - Django - PostgresSQL - AWS*

**Lead Developer**, EventBoard, Salt Lake City, Utah - **2015 -- 2016**

- Django web development.
- Designed and built user management and permissions systems.
- Designed API documentation system using Swagger.
- Increase database migration stability.
- Manage development teams.
- *Python - Django - PostgresSQL - AWS*

**Lead Developer**, JobDash, Salt Lake City, Utah - **2013 -- 2015**

- AngularJS and Django web development.
- Designed and built an analytics system in PostgreSQL and MongoDB.
- Manage and implement development processes for a team of developers.
- Manage server uptime and deployment on AWS and DigitalOcean with Fabric.
- Decrease page load times by a factor of 10.
- *Python - Django - AngularJS - PostgreSQL - MongoDB - AWS - LaTeX*

**Software developer**, SAIC, Tucson, Arizona - **2013**

- Researched and implemented new feature detection algorithms in Python and C++.
- Managed the transition of projects from CVS and SVN to Git.
- Implemented hooks to enforce guidelines and workflow procedure.
- Implemented new features for active websites in Django.
- *Python - Django - C++ - Git*

**Web developer**, *Pedagogy in Large Lectures*, UConn, Storrs, Connecticut - **2011 -- 2012**

- Designed, created, and managed the webpage, videos, surveys, and raw data reports
  for the Pedagogy in Large Lectures research project.
- Managed the transition to new developers.
- *PHP - MySQL - Javascript - LaTeX*

**Web developer**, *Mathematics Department*, UConn, Storrs, Connecticut - **2009 -- 2012**

- Created secure forms for administrative use.
- Updated and created new secure forms for student use.
- Implemented basic security for existing web forms.
- Created extensive user and developer documentation.
- *PHP - MySQL - Javascript*

---
#### Open Source

- [**crit**](https://github.com/LucasRoesler/crit) - review loop between a developer and a coding agent, in Go.
- [**psst**](https://github.com/LucasRoesler/psst) - secrets manager that injects credentials into agent commands without exposing the values.
- [**openfaas-loki**](https://github.com/LucasRoesler/openfaas-loki) - Loki log provider for OpenFaaS.

---
#### Talks / Publications
**Event Driven Postgres with OpenFaas**, Livestream with Alex Elis - **2021-02-02**
> We discuss how you can combine the Postgres WAL, NATS, and OpenFaaS to create a lightweight event driven architecture. Including a short demo of a simple "event echo" to prove the concept. https://www.youtube.com/watch?v=nGOZLOjg5Zs


**Merging Quickly in a Cloud Native World**, Kubecon, Barcelona - **2019**
- Recording: https://youtu.be/d0E7PSlkBbs and Slides: https://static.sched.com/hosted_files/kccnceu19/e2/merging-quickly.pdf
- Summary: https://kccnceu19.sched.com/event/MPZM/merging-quickly-in-a-cloud-native-world-lucas-roesler-contiamo

**OpenFaaS Blog**
- [*Time travelling through your OpenFaaS Function logs*](https://www.openfaas.com/blog/openfaas-loki-into/) - **2020-06-06**
- [*How to Serve Data Science Models with OpenFaaS*](https://www.openfaas.com/blog/python-datascience-with-openfaas/) - **2020-06-04**
- [*How to split large Python Functions across multiple files*](https://www.openfaas.com/blog/multifile-python-functions/) - **2019-01-29**

---
#### Other Experience
* Mathematical research (2007--2013), see https://arxiv.org/search/?query=David-Roesler&searchtype=author&abstracts=show&order=-announced_date_first
* Teaching college mathematics (2007--2013): Levanon Valley College and University of Connecticut

---

#### Interests
* Rock climbing/bouldering, home brewing beer, music, hiking, reading, tabletop gaming, and participating in open source development.
