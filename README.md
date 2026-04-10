# Audit Reports

A collection of smart contract security audit reports and vulnerability findings.

## Repository Structure

| Directory | Protocol | Key Findings |
|-----------|----------|--------------|
| `2025-11-merkl/` | Merkl (DistributionCreator & Distributor) | QA report: event logic bugs, missing validations, governance risks |
| `2025-11-sukukfi/` | SukukFi (ERC-7575 Vault) | Critical missing authorization in withdrawals, QA findings |
| `2025-12-monolith-stablecoin-factory-Hsnmughal/` | Monolith Stablecoin Factory | Interest overflow causing silent loss of accrued interest |
| `tokenize.it-smart-contracts/` | Tokenize.it | Fee zeroing via unguarded `executeFeeChange()` |

## Confidentiality

These reports document security findings for third-party protocols. Please respect any disclosure timelines and do not share findings for protocols that have not yet patched the reported vulnerabilities.

## Author

Hassan Mughal ([@Hsnmughal](https://github.com/Hsnmughal))
