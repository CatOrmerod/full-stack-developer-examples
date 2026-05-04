# Full Stack Developer Examples

Architecture and feature documentation from production work 
on CartAlchemy — a B2B/B2C e-commerce platform built on 
Node.js, Express, Vue.js, MongoDB, and Handlebars.

---

## Blog module
**CartAlchemy V1 · Node.js · Express · MongoDB · S3 · 
CKEditor 5 · Sharp**

Inherited a largely non-functional blog module and led a 
full overhaul to production-ready. Built a dual-mode 
content authoring system (WYSIWYG and page builder), an 
image processing pipeline (Sharp → WebP → S3), scheduled 
publishing, draft preview, and a custom SEO and 
readability engine with 21 checks including Flesch-Kincaid 
scoring.

[Read the architecture →](./blog-architecture.md)

---

## Scheduled orders
**CartAlchemy V1 · Node.js · IBM i (AS400) · DB2/400 · 
JT400 · MongoDB**

End-to-end delivery of a recurring orders feature for a 
B2B education sector platform. The standout technical 
challenge: all schedule data lives in IBM i (AS400) DB2/400 
tables. CartAlchemy communicates with IBM i exclusively 
through a custom JT400 Java bridge, calling stored 
procedures and executing SQL directly against DB2/400. 
MongoDB's role is narrow — mirroring a subset of held 
order data via a cron sync job to support UI display.

[Read the architecture →](./scheduled-orders-architecture.md)
