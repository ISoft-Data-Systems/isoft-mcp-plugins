# ITrack Enterprise Environment Reference

> Source: ITrack Enterprise Environment Diagram (internal ISoft document, shareable with customers)
> This describes the full possible ecosystem. Not every organization uses every component.

---

## Major Required Components

### Enterprise SQL Database
The organization's main database. May be hosted locally on-prem or by ISoft on a Google Cloud VM. ISoft prefers to host for ease of support, but some organizations have security policies requiring on-prem hosting.

Some applications read/write directly to the database. Newer applications use the Enterprise API as a middleware layer. For example:
- **Enterprise Web** communicates solely through the API
- **Enterprise Desktop** talks directly to the database

### Enterprise Desktop Application
The Windows desktop client used for day-to-day business operations. Written in C++, reads/writes directly to the database.

### Crystal Reports and ODBC
Used to generate reports in the desktop application. Crystal Reports is installed on each client machine alongside the desktop app (not shown separately in the diagram). When a report is generated, the desktop app requests it from Crystal, which returns a PDF. Crystal connects directly to the Enterprise Database via ODBC (Microsoft's Open Database Connectivity interface).

- One-off desktop printing: no additional infrastructure needed
- Printing from web applications: requires a Crystal Server
- Scheduled print jobs: requires Report Queue

### Enterprise API
The central API layer for accessing Enterprise data. Required for Enterprise Web and some other features. Usually hosted on the same server as the database, but not necessarily (e.g., cloud-hosted API accessing on-prem data is possible with IT coordination).

APIs protect the database schema from direct external access and provide versioning protection so downstream connections don't break when the schema changes.

---

## Recommended Optional Components

### Enterprise Web Application (EEW)
A web-based UI for the Enterprise platform. Communicates with the Enterprise database via the Enterprise API. Supports many of the same workflows as the desktop app, but not all — it is currently primarily a companion to the desktop app, with the goal of eventually supporting all daily workflows independently.

- Supported in all browsers and devices; responsive for mobile and desktop
- Report generation requires a Crystal Server
- Usually deployed on ISoft's cloud VMs; on-prem deployment requires Docker and Node 20

### Crystal Server and Report Commander
Required for report generation in the web application. A Windows machine must be configured as a Crystal Server. Organizations can use ISoft's Crystal Server or set up their own on-prem.

When a user prints or previews a report in EEW:
1. Enterprise API sends report parameters to Crystal Server
2. Crystal Server connects to the database to generate the report
3. Crystal Server sends a command to Report Commander
4. Report Commander connects to the database via ODBC and generates the PDF
5. PDF is served to the user's browser

### Report Queue and Report Commander
Handles automated print job routing in an organization's internal network. Not required for basic printing — only needed when:
- Print jobs must be routed to specific printers automatically (e.g., tags to a tag printer, invoices to front desk, picklists to warehouse)
- Scheduled email reports are needed
- A user wants to print from EEW directly to an internal printer without configuring it on their local machine

If only email scheduling is needed (not printer routing), ISoft can configure a Report Queue instance on their internal servers.

Report Queue is a Windows application. It monitors the `reportqueue` table in the database for new jobs, then uses Report Commander (via ODBC) to generate PDFs. Email jobs are handled by Report Queue after PDF generation.

---

## Websites and eCommerce

### HeavyTruckParts.Net (HTP)
A public-facing eCommerce site for heavy truck parts, available to all ITrack users regardless of platform. Data is synced from each organization's database to HTP's replication and read databases by the Aggregator.

### Aggregator and HTP Databases
Aggregator is middleware that syncs an organization's inventory data to the HTP read and replication databases. Primarily serves HeavyTruckParts.Net, advertising websites, and customer websites.

### HTP API
A dedicated API built for 3rd party websites to access the same inventory data as HTP. Used by organizations that want to feed inventory to their own website (ISoft-built or 3rd party). Has per-organization access controls.

**HTP API vs. Enterprise API:**
- **HTP API** — built for eCommerce; smaller data set; purpose-built for displaying sellable inventory to customers; hides complex pricing and availability rules
- **Enterprise API** — built for employee-facing applications; full data access; used for supplemental or alternative UIs for internal workflows
- In short: HTP API is for an organization's **customers**, Enterprise API is for an organization's **employees**

### Custom Website Development
ISoft can build custom WordPress websites for customers, pulling data through the HTP API.

### 3rd Party Website Development
Organizations can be granted HTP API access for their own developers or contractors to build their own website. Development is the organization's responsibility.

### Scheduled Export Feeds and Advertising Partners
ISoft can configure automated nightly data exports to advertising partners (e.g., TPI / Truck Parts Inventory). Data is pulled from the HTP database. Some partners use the HTP API directly (e.g., MLS / My Little Salesman).

---

## Additional Features

### ITrack LX
A lightweight web application for specialized operational functions: warehouse management, barcode scanning, employee time tracking, and quoting. Hosted on the same server as the database and communicates directly with it.

### Enterprise MCP
A configured plugin that exposes Enterprise API functionality to AI agents. Agents connect to the organization's data and interact using natural language. The agent logs in with the same credentials and permissions as the user. Requires the Enterprise API.

### Desktop Integrations
The desktop application supports integrations including VIN decoding, eBay auctions, UPS Worldship, QuickBooks, and others.

### API Integrations
Some integrations use the Enterprise API rather than the desktop app, such as tax service integrations (TaxJar, DSTax).

### 3rd Party App Development
The Enterprise API allows 3rd party developers to build custom applications using an organization's Enterprise data. Access must be granted by an org admin; ISoft recommends 3rd party developers communicate with ISoft for technical guidance. Development is the organization's and developer's responsibility.
