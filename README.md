# 🚀 Polar Semiconductor SQL & Data Architecture Documentation

This repository contains comprehensive architectural, data lineage, and entity-relationship documentation for database objects, views, and functions within the **Polar Semiconductor (VisionProd)** manufacturing execution and data warehouse environment.

---

## 📚 Documentation Index

| File | Object Name | Object Type | Description |
| :--- | :--- | :---: | :--- |
| 📄 **[view1.md](view1.md)** | `dbo.LOTHISTV` | `SQL View` | Enriched WIP lot history and event tracking view. Includes complete column dictionary, ER diagram, data lineage flowchart, and filter logic. |
| 📄 **[view2.md](view2.md)** | `dbo.PF_LOTHIST_HISTORY3` | `Scalar UDF` | Historical lot comment decoding engine with 60+ transaction history codes, positional parsing rules, and performance optimization details. |

---

## 🛠️ Key Architectural Components Documented

- **Data Lineage Flowcharts**: Visualizing flows across Database Roles, Warehouses/Compute, Source Tables, Transformations, and Consuming Layers using Mermaid diagrams.
- **Entity-Relationship (ER) Diagrams**: Complete structural relationship definitions and cardinality between base tables, lookup tables, functions, and views.
- **Business Logic Catalogs**: Granular parsing rules, hold status flags, queue milestones, and operator classification logic.
