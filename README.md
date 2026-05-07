# Global Employment Platform (EOR)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source Employer of Record platform that enables companies to compliantly hire and pay employees in 80+ countries without establishing local legal entities.

A Global Employment Platform handles the full lifecycle of international employment: locally compliant contracts, multi-currency payroll, statutory benefits enrolment, tax withholding, and ongoing regulatory compliance. It serves companies that want to hire talent anywhere in the world without the $20,000-$50,000 cost and 6-18 month timeline of setting up a foreign subsidiary. The platform acts as the legal employer of record in each jurisdiction while the client company retains day-to-day management of the worker.

---

## Why Global Employment Platform (EOR)?

- **Opaque, high-cost pricing.** Incumbents like Deel and Remote charge $499-$699 per employee per month. Enterprise players like Globalization Partners and Safeguard Global use entirely custom pricing with no public rate cards, making cost comparison difficult and budget planning unreliable.

- **No open-source alternative exists.** Every EOR platform on the market is proprietary commercial SaaS. The compliance engines, contract generators, and payroll logic are black boxes, making it impossible for organisations to audit, extend, or self-host critical employment infrastructure.

- **Compliance monitoring is reactive, not predictive.** Most platforms alert when a law has already changed. Few offer forecasting of upcoming regulatory impacts or automated financial modelling of how changes affect payroll costs across jurisdictions.

- **Equity administration is an afterthought.** International equity grants with correct cross-border tax treatment are a high-value use case, yet almost no EOR platform integrates meaningfully with equity administration platforms like Carta or Shareworks.

- **Contractor-to-employee conversion is underexecuted.** With aggressive misclassification enforcement in the EU, UK, and Brazil, companies need structured conversion pathways. Most platforms offer basic classification questionnaires but lack end-to-end conversion workflows with risk scoring.

---

## Key Features

### Compliant Employment and Contracts

- Automated generation of locally compliant employment agreements based on jurisdiction, role, compensation structure, and applicable collective bargaining agreements
- E-signature workflow with centralised storage and version control tracking regulatory changes requiring contract amendments
- Contractor-to-employee conversion tools with misclassification risk assessment questionnaires
- Permanent establishment (PE) risk management to prevent PE liability creation

### Payroll and Compensation

- Multi-currency payroll processing with correct statutory deductions (income tax, social security, pension contributions, health insurance)
- Automated tax filings and statutory reports submitted to local authorities
- Real-time financial analytics showing labour costs and spend by country, currency, and contract type
- Payroll accuracy verification using ML to flag anomalies and potential errors before submission

### Benefits Administration

- Automatic enrolment in mandatory benefits (national health insurance, pension schemes, leave entitlements) per jurisdiction
- Supplemental benefits configuration (private health insurance, life insurance, wellness allowances)
- AI-driven benefit optimisation recommending benefit packages based on employee circumstances to maximise satisfaction and retention
- Cross-border benefits comparison tools for cost-effective package design

### Compliance and Regulatory Monitoring

- Continuous tracking of labour law changes, tax rate updates, minimum wage adjustments across all active jurisdictions
- Predictive compliance alerts forecasting regulatory impacts 60+ days before effective dates
- Real-time compliance impact analysis showing financial effects of changes (e.g., "new minimum wage increases labour costs by X% in Y country")
- AI-powered contract amendment automation when regulatory changes require updates

### Onboarding, Offboarding, and Workforce Management

- Digitised document collection (identity verification, right-to-work checks, tax forms) with background check coordination
- Compliant offboarding with jurisdiction-specific notice period management, severance calculation, and final pay processing
- Visa and mobility support for immigration-active countries
- Integration with major HRIS platforms (Workday, BambooHR, HiBob), ATS platforms (Greenhouse, Lever), and accounting software (Xero, QuickBooks)

---

## AI-Native Advantage

The current generation of EOR platforms bolted AI onto existing rule-based compliance engines. An AI-native approach changes the architecture fundamentally: ML models predict regulatory changes and their financial impact before they take effect, enabling proactive rather than reactive compliance. AI analyses job descriptions, work patterns, and control relationships to score misclassification probability before classification decisions are made. Payroll runs are reviewed by anomaly detection models that catch errors traditional rule engines miss. Compensation structures are analysed to identify tax-efficient approaches within legal bounds, including navigating complex international tax treaties to minimise withholding and double-taxation risk.

---

## Tech Stack & Deployment

The platform targets hybrid deployment: a managed cloud offering for organisations that want turnkey EOR services, and self-hosted components for enterprises that need to keep employment data within their own infrastructure. Integration follows an API-first approach with native connectors to major HRIS/HCM platforms (Workday, SAP SuccessFactors, BambooHR, HiBob), ATS platforms (Greenhouse, Lever), and accounting systems (Xero, QuickBooks, Sage Intacct). The compliance engine requires a structured legal entity network (owned or via verified partners) in each supported jurisdiction.

---

## Market Context

The global EOR market reached approximately $6.82 billion in 2025 and is growing at around 9% annually. The market is led by Deel (150+ countries, 35,000+ customers), Remote (90+ owned entities), Oyster HR (180+ countries), and Globalization Partners (180+ countries). Per-employee pricing ranges from $499 to $699/month at the mid-market level, with enterprise pricing entirely custom. Primary buyers are mid-market and enterprise companies hiring remote talent internationally, though there is an underserved segment of SMBs with 5-50 employees who find current platforms over-engineered and overpriced.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
