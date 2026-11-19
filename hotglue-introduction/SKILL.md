---
name: hotglue-introduction
description: "Hotglue crash-course for agents"
---

## What is hotglue?

hotglue is an embedded iPaaS for integrating your product with ERPs, Adtech platforms, CRMs, databases, and more.

hotglue is **embeddable**, **code-first**, and built on the **Python ecosystem** — enabling developers to build flexible, scalabale integrations without the lock-in of traditional iPaaS tools.

## Definitions

### Tenants

A **tenant** is one of your customers who has connected at least one integration via hotglue.

Every environment starts with a default tenant, which contains the default configuration for connectors, catalogs, and fieldmaps.

### Flows

A **flow** groups a type of integration you offer and the connectors users can connect within it.

### Available Connectors

An **available connector** is any connector in your environment that you can enable on a flow. Within the available connector configurations is information about how tenants link (ie api keys, OAuth urls), how they appear to your tenants (Name, Icon, etc), and what version of the connector gets used in jobs.

### Supported Connectors

A **supported connector** is a connector you have enabled for a specific flow so tenants can link it. Supported connectors can contain configurations that get used for all tenants who link to that flow, such as a Client ID+Secret for an OAuth app.

### Linked Connectors

A **linked connector** is a connector a tenant has authenticated and connected on a flow. Once a tenant links they can also have:
- A catalog -> The list of tables and fields available to sync for that platform
- A fieldmap -> The selection of tables and fields that will be synced for this tenant in jobs


## How hotglue is used end to end

### 1) Tenants link their credentials

This can be done via the hotglue widget or a magic link. The magic link is a short-lived authenticated url that can be sent to tenants so they can pick their connector and enter their credentials. For agent-driven OAuth linking, use `authenticating-oauth`.

### 2) Run a discover

For platforms where tenants might have custom objects and fields, you can run a "discover" for that tenant's linked connector. This will kick off an asyncronous task to discover what tables and fields are available. The discover produces a catalog of these tables.

If you do not run a discover for a tenant, they will use the default tenant's catalog.

### 3) Select field map

Once the catalog is generated, you can set the "field map" for the tenant. This determines what tables and fields will by synced

### 4) Run a job

Hotglue jobs have 3 steps:

- Sync:
  - This pulls data from the source system based on the field map.
  - The source system is usually interacted with via an open source singer tap
  - The raw singer output of the tap is converted into csv or parquet data before the transform script runs
- ETL(Transform):
  - This custom script maps the sync output into the export format
  - Hotglue customers can upload custom transformation scripts to perform logic like mapping, joins, filters, etc
- Export:
  - This writes the ETL output into the target system
 
