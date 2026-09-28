# Graph Report - ERP  (2026-09-28)

## Corpus Check
- 396 files · ~360,004 words
- Verdict: corpus is large enough that graph structure adds value.
- Unclassified: 11 file(s) not represented in the graph (top: (none) 8, .example 1, .css 1)

## Summary
- 2683 nodes · 9669 edges · 126 communities (87 shown, 39 thin omitted)
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS · INFERRED: 9 edges (avg confidence: 0.85)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- Frontend Shared UI & Document Engine
- Frontend Shared UI & Document Engine
- Purchasing & Procurement
- Domain Subsystem 3
- Domain Subsystem 4
- Purchasing & Procurement
- Purchasing & Procurement
- HRIS & Indonesian Statutory Payroll
- Sales & Customer Distribution
- Database Schema & Persistence
- HRIS & Indonesian Statutory Payroll
- HRIS & Indonesian Statutory Payroll
- Frontend Shared UI & Document Engine
- IAM & Platform Infrastructure
- Sales & Customer Distribution
- Purchasing & Procurement
- IAM & Platform Infrastructure
- Domain Subsystem 17
- Domain Subsystem 18
- Manufacturing Execution & Job Costing
- Sales & Customer Distribution
- Purchasing & Procurement
- Sales & Customer Distribution
- Purchasing & Procurement
- Accounting & Financial Ledger
- IAM & Platform Infrastructure
- Sales & Customer Distribution
- Accounting & Financial Ledger
- IAM & Platform Infrastructure
- Inventory & Warehouse Movements
- Domain Subsystem 30
- Purchasing & Procurement
- Sales & Customer Distribution
- Purchasing & Procurement
- Sales & Customer Distribution
- Quality Assurance & ISO Control
- Frontend Shared UI & Document Engine
- Purchasing & Procurement
- Frontend Shared UI & Document Engine
- Sales & Customer Distribution
- IAM & Platform Infrastructure
- Sales & Customer Distribution
- Purchasing & Procurement
- Sales & Customer Distribution
- Quality Assurance & ISO Control
- Sales & Customer Distribution
- Domain Subsystem 46
- Sales & Customer Distribution
- Accounting & Financial Ledger
- Purchasing & Procurement
- Purchasing & Procurement
- Purchasing & Procurement
- Purchasing & Procurement
- Domain Subsystem 53
- Frontend Shared UI & Document Engine
- Inventory & Warehouse Movements
- Sales & Customer Distribution
- Purchasing & Procurement
- Purchasing & Procurement
- Sales & Customer Distribution
- Sales & Customer Distribution
- Purchasing & Procurement
- Purchasing & Procurement
- Purchasing & Procurement
- Inventory & Warehouse Movements
- Accounting & Financial Ledger
- Domain Subsystem 66
- Purchasing & Procurement
- IAM & Platform Infrastructure
- Purchasing & Procurement
- Domain Subsystem 70
- Purchasing & Procurement
- Domain Subsystem 72
- Purchasing & Procurement
- Purchasing & Procurement
- Sales & Customer Distribution
- Purchasing & Procurement
- Domain Subsystem 77
- Purchasing & Procurement
- IAM & Platform Infrastructure
- Sales & Customer Distribution
- Sales & Customer Distribution
- Sales & Customer Distribution
- Quality Assurance & ISO Control
- Sales & Customer Distribution
- Purchasing & Procurement
- Treasury & Bank Reconciliation
- Domain Subsystem 87
- Purchasing & Procurement
- Purchasing & Procurement
- Inventory & Warehouse Movements
- Manufacturing Execution & Job Costing
- Sales & Customer Distribution
- Domain Subsystem 93
- Domain Subsystem 94
- Inventory & Warehouse Movements
- Domain Subsystem 96
- Domain Subsystem 97
- Purchasing & Procurement
- Sales & Customer Distribution
- Frontend Shared UI & Document Engine
- Inventory & Warehouse Movements
- Purchasing & Procurement
- Domain Subsystem 103
- Domain Subsystem 104
- Domain Subsystem 105
- Purchasing & Procurement
- Purchasing & Procurement
- Domain Subsystem 108
- Domain Subsystem 109
- Purchasing & Procurement
- IAM & Platform Infrastructure
- Inventory & Warehouse Movements
- Sales & Customer Distribution
- Domain Subsystem 114
- Domain Subsystem 115
- Domain Subsystem 116
- Domain Subsystem 117
- Sales & Customer Distribution
- Domain Subsystem 120
- Domain Subsystem 121
- Database Schema & Persistence
- Domain Subsystem 124

## God Nodes (most connected - your core abstractions)
1. `Success()` - 244 edges
2. `react` - 187 edges
3. `lucide-react` - 177 edges
4. `formatCurrency()` - 143 edges
5. `Button` - 119 edges
6. `useAuthStore` - 101 edges
7. `api` - 92 edges
8. `DataTable()` - 91 edges
9. `cn()` - 91 edges
10. `sonner` - 89 edges

## Surprising Connections (you probably didn't know these)
- `PopoverContent` --calls--> `cn()`  [EXTRACTED]
  apps/web/components/ui/popover.tsx → apps/web/lib/utils.ts
- `TestGenerateAndValidateToken()` --calls--> `GenerateAccessToken()`  [INFERRED]
  services/api/internal/platform/middleware/auth_test.go → services/api/internal/platform/middleware/auth.go
- `RequirePermission()` --calls--> `ExtractClientIP()`  [INFERRED]
  services/api/internal/platform/middleware/auth.go → services/api/internal/platform/middleware/ratelimit.go
- `LoginPage()` --calls--> `Badge()`  [EXTRACTED]
  apps/web/app/(auth)/login/page.tsx → apps/web/components/ui/badge.tsx
- `LoginPage()` --calls--> `useAuthStore`  [EXTRACTED]
  apps/web/app/(auth)/login/page.tsx → apps/web/stores/auth-store.ts

## Import Cycles
- None detected.

## Communities (126 total, 39 thin omitted)

### Community 0 - "Frontend Shared UI & Document Engine"
Cohesion: 0.05
Nodes (179): AuditActivityStream(), AssetsRegistryTab(), AssetsRegistryTabProps, DepreciationTab(), DepreciationTabProps, DisposalTab(), DisposalTabProps, buildCreateAssetPayload() (+171 more)

### Community 1 - "Frontend Shared UI & Document Engine"
Cohesion: 0.07
Nodes (63): CATEGORY_TABS, LoginFormValues, LoginPage(), loginSchema, ROLE_PRESETS, RolePreset, DebitNotesTab(), DebitNotesTabProps (+55 more)

### Community 2 - "Purchasing & Procurement"
Cohesion: 0.08
Nodes (36): HoldingBoardProps, OperationalHighlightsProps, VendorBillDetailPage(), PurchaseOrderDetailPage(), PurchaseRequisitionDetailPage(), DeliveryOrderDetailPage(), SalesInvoiceDetailPage(), SalesOrderDetailPage() (+28 more)

### Community 3 - "Domain Subsystem 3"
Cohesion: 0.06
Nodes (7): net/http.Request, Handler, Handler, Handler, GetBranchID(), KeysetPaginated(), ParseKeysetParams()

### Community 4 - "Domain Subsystem 4"
Cohesion: 0.11
Nodes (24): go_pkg_context, go_pkg_crypto_rand, go_pkg_crypto_sha256, go_pkg_embed, go_pkg_encoding_hex, go_pkg_errors, go_pkg_fmt, go_pkg_github_com_aws_aws_sdk_go_v2_aws (+16 more)

### Community 5 - "Purchasing & Procurement"
Cohesion: 0.10
Nodes (49): WarehousesTab(), BinsTab(), BinsTabProps, BranchesTab(), BranchesTabProps, ItemsTab(), ItemsTabProps, OrgTab() (+41 more)

### Community 6 - "Purchasing & Procurement"
Cohesion: 0.05
Nodes (50): go_pkg_bytes, go_pkg_encoding_csv, go_pkg_testing, testing.T, CalculateDisposalGainLoss(), CalculateStraightLineMonthlyDepreciation(), TestFixedAssets_DisposalGainLoss(), TestFixedAssets_StraightLineFormulaPSAK16() (+42 more)

### Community 7 - "HRIS & Indonesian Statutory Payroll"
Cohesion: 0.10
Nodes (44): AttendanceTab(), AttendanceTabProps, EmployeesTab(), EmployeesTabProps, LeavesTab(), LeavesTabProps, PayrollTab(), PayrollTabProps (+36 more)

### Community 8 - "Sales & Customer Distribution"
Cohesion: 0.05
Nodes (59): Entry, Recorder, Handler, github.com/jackc/pgx/v5/pgxpool.Pool, main(), Service, NewHandler(), NewRepository() (+51 more)

### Community 9 - "Database Schema & Persistence"
Cohesion: 0.07
Nodes (27): ApprovalHistory, ApprovalDelegation, ApprovalRequest, pgx.Row, pgx.Tx, insertHistoryTx(), scanRequestRow(), syncDocumentRevisionTx() (+19 more)

### Community 10 - "HRIS & Indonesian Statutory Payroll"
Cohesion: 0.06
Nodes (25): AttendanceRecord, BiometricPunchLog, CheckInInput, CheckOutInput, CreateEmployeeInput, CreateLeaveInput, Employee, LeaveRequest (+17 more)

### Community 11 - "HRIS & Indonesian Statutory Payroll"
Cohesion: 0.07
Nodes (3): Handler, Handler, Success()

### Community 12 - "Frontend Shared UI & Document Engine"
Cohesion: 0.12
Nodes (34): ArchiveTab(), ArchiveTabProps, DocumentEditorTab(), DocumentEditorTabProps, TemplatesTab(), TemplatesTabProps, DocumentsHubPage(), DOCUMENT_TEMPLATES (+26 more)

### Community 13 - "IAM & Platform Infrastructure"
Cohesion: 0.20
Nodes (21): go_pkg_encoding_base64, go_pkg_encoding_json, go_pkg_github_com_go_chi_chi_v5, go_pkg_github_com_golang_jwt_jwt_v5, go_pkg_net_http, go_pkg_net_url, go_pkg_path_filepath, go_pkg_strconv (+13 more)

### Community 14 - "Sales & Customer Distribution"
Cohesion: 0.08
Nodes (5): net/http.ResponseWriter, Handler, Service, NewHandler(), Handler

### Community 15 - "Purchasing & Procurement"
Cohesion: 0.12
Nodes (32): AuditTab(), BranchesTab(), BranchesTabProps, CompanyTabProps, FiscalTab(), FiscalTabProps, InfrastructureTab(), IntegrationsTab() (+24 more)

### Community 16 - "IAM & Platform Infrastructure"
Cohesion: 0.08
Nodes (22): AuditLogItem, AuditorUserView, AuthResult, CreateUserInput, LoginInput, PermissionItem, RoleItem, Service (+14 more)

### Community 17 - "Domain Subsystem 17"
Cohesion: 0.07
Nodes (23): net/http.Handler, chi.Router, chi.Router, chi.Router, chi.Router, chi.Router, chi.Router, chi.Router (+15 more)

### Community 18 - "Domain Subsystem 18"
Cohesion: 0.08
Nodes (33): AlertLevel, AlertPayload, Dispatcher, go_pkg_bufio, go_pkg_net, go_pkg_net_http_httptest, go_pkg_runtime, go_pkg_sync (+25 more)

### Community 19 - "Manufacturing Execution & Job Costing"
Cohesion: 0.15
Nodes (28): BOMTab(), BOMTabProps, CostingTab(), CostingTabProps, IssuesTab(), IssuesTabProps, MRPTab(), MRPTabProps (+20 more)

### Community 20 - "Sales & Customer Distribution"
Cohesion: 0.08
Nodes (9): ReceivePOInput, ReceivePOItemInput, Repository, UpdateVendorInput, PurchaseOrder, PurchaseRequisition, RFQ, Vendor (+1 more)

### Community 21 - "Purchasing & Procurement"
Cohesion: 0.16
Nodes (29): AdjustmentsTab(), AdjustmentsTabProps, BinsTab(), BinsTabProps, ItemsTab(), ItemsTabProps, LotsTab(), LotsTabProps (+21 more)

### Community 22 - "Sales & Customer Distribution"
Cohesion: 0.13
Nodes (28): AgingTab(), AgingTabProps, CustomersTabProps, InvoicesTab(), InvoicesTabProps, OrdersTab(), OrdersTabProps, PickingTab() (+20 more)

### Community 23 - "Purchasing & Procurement"
Cohesion: 0.10
Nodes (7): context.Context, Repository, roleWithPerms, userRow, AuditLogItem, RoleItem, UserBranchAccess

### Community 24 - "Accounting & Financial Ledger"
Cohesion: 0.18
Nodes (26): BankReconciliationTab(), BankReconciliationTabProps, COATab(), COATabProps, JournalsTab(), JournalsTabProps, MultiCurrencyTab(), MultiCurrencyTabProps (+18 more)

### Community 25 - "IAM & Platform Infrastructure"
Cohesion: 0.06
Nodes (34): dependencies, class-variance-authority, clsx, @hookform/resolvers, html2canvas, jspdf, jszip, lucide-react (+26 more)

### Community 26 - "Sales & Customer Distribution"
Cohesion: 0.09
Nodes (18): CompleteWorkOrderInput, CreateAssetInput, CreatePartConsumptionItem, CreateScheduleInput, CreateWorkOrderInput, MaintenanceAsset, MaintenancePartConsumption, MaintenanceSchedule (+10 more)

### Community 27 - "Accounting & Financial Ledger"
Cohesion: 0.08
Nodes (21): Account, AccountBalanceItem, BalanceSheetReport, CreateJournalInput, CreateLineInput, IncomeStatementReport, JournalEntry, JournalLine (+13 more)

### Community 28 - "IAM & Platform Infrastructure"
Cohesion: 0.06
Nodes (31): name, private, version, autoprefixer, clsx, eslint, eslint-config-next, html2canvas (+23 more)

### Community 29 - "Inventory & Warehouse Movements"
Cohesion: 0.09
Nodes (11): Service, BusinessPartner, CompanyProfile, DocumentSeries, FiscalYear, ItemMaster, Repository, SecurityPolicy (+3 more)

### Community 30 - "Domain Subsystem 30"
Cohesion: 0.17
Nodes (22): AuditActivityStreamProps, DepartmentSwitcher(), DepartmentSwitcherProps, HoldingBoard(), KpiMetricCards(), KpiMetricCardsProps, OperationalHighlights(), WelcomeBanner() (+14 more)

### Community 31 - "Purchasing & Procurement"
Cohesion: 0.09
Nodes (9): Repository, BusinessPartner, CompanyProfile, DocumentSeries, FiscalYear, ItemMaster, SecurityPolicy, Warehouse (+1 more)

### Community 32 - "Sales & Customer Distribution"
Cohesion: 0.09
Nodes (16): CreatePurchaseReturnInput, CreateSalesReturnInput, PurchaseReturn, PurchaseReturnItem, Repository, SalesReturn, SalesReturnItem, Service (+8 more)

### Community 33 - "Purchasing & Procurement"
Cohesion: 0.10
Nodes (14): PettyCashVoucher, PettyCashVoucher, Repository, NewService(), TestPettyCashValidation(), BankStatement, BankStatementLine, CreateBankStatementInput (+6 more)

### Community 34 - "Sales & Customer Distribution"
Cohesion: 0.12
Nodes (13): CreateDeliveryProofInput, CreateTripOrderInput, CreateVehicleInput, DeliveryProof, FleetVehicle, Repository, Service, SettleTripExpenseInput (+5 more)

### Community 35 - "Quality Assurance & ISO Control"
Cohesion: 0.12
Nodes (13): CertificateOfAnalysis, CreateCoaInput, CreateInspectionInput, CreateInspectionItemIn, CreateNCRInput, NCRTicket, QCInspection, QCInspectionItem (+5 more)

### Community 36 - "Frontend Shared UI & Document Engine"
Cohesion: 0.18
Nodes (21): ApprovalDetailModal(), ApprovalDetailModalProps, DelegationsTab(), DelegationsTabProps, RequestsTab(), RequestsTabProps, WorkflowPoliciesTab(), WorkflowPoliciesTabProps (+13 more)

### Community 37 - "Purchasing & Procurement"
Cohesion: 0.11
Nodes (14): AssetDisposal, CreateDisposalInput, CreateFixedAssetInput, DepreciationBatch, FixedAsset, PostDepreciationInput, Repository, Service (+6 more)

### Community 38 - "Frontend Shared UI & Document Engine"
Cohesion: 0.16
Nodes (21): go_pkg_database_sql, go_pkg_flag, go_pkg_github_com_jackc_pgx_v5_stdlib, go_pkg_github_com_pressly_goose_v3, go_pkg_log, go_pkg_os, BaselineConfig, BaselineResult (+13 more)

### Community 39 - "Sales & Customer Distribution"
Cohesion: 0.11
Nodes (7): AwardBidInput, CreatePOInput, CreatePOItemInput, Service, UpdatePOInput, PurchaseOrder, VendorDebitNote

### Community 40 - "IAM & Platform Infrastructure"
Cohesion: 0.14
Nodes (5): Service, NewHandler(), resolveActorName(), ExtractClientIP(), Handler

### Community 41 - "Sales & Customer Distribution"
Cohesion: 0.12
Nodes (9): CreateInvoiceInput, CreateInvoiceItemInput, Service, UpdateCustomerInput, Customer, Repository, SalesInvoice, SalesQuotation (+1 more)

### Community 42 - "Purchasing & Procurement"
Cohesion: 0.13
Nodes (12): NSFPAllocation, TaxInvoice, NSFPAllocation, Repository, TaxInvoice, NewService(), CreateNSFPInput, CreateTaxInvoiceInput (+4 more)

### Community 43 - "Sales & Customer Distribution"
Cohesion: 0.16
Nodes (5): CreateCustomerInput, Repository, Customer, SalesInvoice, SalesOrder

### Community 44 - "Quality Assurance & ISO Control"
Cohesion: 0.11
Nodes (18): compilerOptions, allowJs, esModuleInterop, incremental, isolatedModules, jsx, lib, module (+10 more)

### Community 45 - "Sales & Customer Distribution"
Cohesion: 0.14
Nodes (14): CreateSQInput, CreateSQItemInput, Customer, CustomerStatementDTO, PickingSlip, PickingSlipItem, ReleaseCreditInput, SalesInvoice (+6 more)

### Community 46 - "Domain Subsystem 46"
Cohesion: 0.14
Nodes (11): go_pkg_github_com_hibiken_asynq, github.com/hibiken/asynq.Client, github.com/hibiken/asynq.ServeMux, github.com/hibiken/asynq.Server, github.com/hibiken/asynq.Task, github.com/hibiken/asynq.TaskInfo, Client, Server (+3 more)

### Community 47 - "Sales & Customer Distribution"
Cohesion: 0.14
Nodes (9): Service, CompleteWorkOrderInput, CreateWorkOrderInput, MaterialIssue, Repository, WorkCenter, WorkOrder, WorkOrderOperation (+1 more)

### Community 48 - "Accounting & Financial Ledger"
Cohesion: 0.15
Nodes (6): Repository, Account, BalanceSheetReport, IncomeStatementReport, JournalEntry, TrialBalanceReport

### Community 49 - "Purchasing & Procurement"
Cohesion: 0.18
Nodes (13): ApprovalTab(), ApprovalTabProps, apps_web_components_layout_notifications_index_approvalrequestitem, apps_web_components_layout_notifications_index_approvaltab, apps_web_components_layout_notifications_index_inventoryalerttab, apps_web_components_layout_notifications_index_inventoryitemalert, apps_web_components_layout_notifications_index_storage_key, InventoryAlertTab() (+5 more)

### Community 50 - "Purchasing & Procurement"
Cohesion: 0.22
Nodes (15): AgingPoint, BranchDistributionPoint, CashflowPoint, CategoryValuationPoint, DashboardCharts, HrmAttendancePoint, HrmDepartmentPoint, InventoryVelocityPoint (+7 more)

### Community 51 - "Purchasing & Procurement"
Cohesion: 0.12
Nodes (15): aliases, components, hooks, lib, ui, utils, rsc, $schema (+7 more)

### Community 52 - "Purchasing & Procurement"
Cohesion: 0.18
Nodes (15): ExecutiveCharts(), FinanceCharts(), HrmCharts(), apps_web_components_dashboard_charts_index_executivecharts, apps_web_components_dashboard_charts_index_financecharts, apps_web_components_dashboard_charts_index_hrmcharts, apps_web_components_dashboard_charts_index_inventorycharts, apps_web_components_dashboard_charts_index_manufacturingcharts (+7 more)

### Community 54 - "Frontend Shared UI & Document Engine"
Cohesion: 0.23
Nodes (11): DashboardLayout(), CommandPalette(), PALETTE_ITEMS, PaletteItem, ALL_BRANCH_OPTION, DEFAULT_BRANCHES, Header(), NotificationDropdown() (+3 more)

### Community 55 - "Inventory & Warehouse Movements"
Cohesion: 0.14
Nodes (7): BinTransferInput, Service, FEFORecommendationResponse, InventoryLot, Repository, StockAdjustment, StockMovement

### Community 56 - "Sales & Customer Distribution"
Cohesion: 0.16
Nodes (12): BOMRouting, CompleteWorkOrderInput, CreateMaterialIssueInput, CreateMaterialIssueItemInput, CreateWorkOrderInput, GeneratePRFromMRPInput, MaterialIssue, MaterialIssueItem (+4 more)

### Community 57 - "Purchasing & Procurement"
Cohesion: 0.21
Nodes (7): ExecutiveSummary, Repository, Service, TestExecutiveSummaryKPIFormulas(), NewRepository(), Repository, NewService()

### Community 58 - "Purchasing & Procurement"
Cohesion: 0.17
Nodes (5): github.com/aws/aws-sdk-go-v2/service/s3.Client, github.com/aws/aws-sdk-go-v2/service/s3.PresignClient, NewStorageHandler(), S3Adapter, NewS3Adapter()

### Community 59 - "Sales & Customer Distribution"
Cohesion: 0.22
Nodes (6): Repository, CompleteWorkOrderInput, CreateWorkOrderInput, MaterialIssue, WorkOrder, WorkOrderOperation

### Community 60 - "Sales & Customer Distribution"
Cohesion: 0.24
Nodes (10): ApprovalHistoryItem, GoodsReceiptItemRecord, GoodsReceiptRecord, PurchaseOrder, PurchaseOrderItem, PurchaseRequisition, PurchaseRequisitionItem, RFQ (+2 more)

### Community 61 - "Purchasing & Procurement"
Cohesion: 0.20
Nodes (5): Repository, NewService(), TestInfrastructureHealthAggregator(), Config, Service

### Community 62 - "Purchasing & Procurement"
Cohesion: 0.24
Nodes (11): time.Time, FEFOAllocatedLot, FEFORecommendationRequest, FEFORecommendationResponse, InventoryLot, StockAdjustment, StockMovement, StockTransfer (+3 more)

### Community 63 - "Purchasing & Procurement"
Cohesion: 0.20
Nodes (5): Repository, FEFORecommendationResponse, InventoryLot, StockMovement, StockTransfer

### Community 64 - "Inventory & Warehouse Movements"
Cohesion: 0.17
Nodes (8): ref_k6, options, errorRate, inventoryLatency, masterDataLatency, options, purchasingLatency, successfulRequests

### Community 67 - "Purchasing & Procurement"
Cohesion: 0.24
Nodes (3): BiometricDevice, CreateDeviceInput, Repository

### Community 68 - "IAM & Platform Infrastructure"
Cohesion: 0.38
Nodes (7): SidebarDepartmentSwitcher(), allNavItems, defaultNavSections, DEPARTMENT_CONFIG, NavItem, NavSection, NavSubItem

### Community 69 - "Purchasing & Procurement"
Cohesion: 0.20
Nodes (10): devDependencies, autoprefixer, eslint, eslint-config-next, postcss, tailwindcss, @types/node, @types/react (+2 more)

### Community 70 - "Domain Subsystem 70"
Cohesion: 0.22
Nodes (3): redis.Client, ValkeyClient, NewValkeyClient()

### Community 71 - "Purchasing & Procurement"
Cohesion: 0.24
Nodes (4): BillOfMaterials, BOMComponent, CreateBOMComponentInput, CreateBOMInput

### Community 73 - "Purchasing & Procurement"
Cohesion: 0.22
Nodes (9): BusinessPartner, CompanyProfile, DocumentSeries, FiscalPeriod, FiscalYear, ItemMaster, SecurityPolicy, Warehouse (+1 more)

### Community 74 - "Purchasing & Procurement"
Cohesion: 0.22
Nodes (4): CreateVendorBillInput, CreateVendorBillItemInput, VendorBill, VendorBillItem

### Community 75 - "Sales & Customer Distribution"
Cohesion: 0.22
Nodes (5): ConvertQuotationInput, CreateSOInput, CreateSOItemInput, UpdateSOInput, SalesOrder

### Community 77 - "Domain Subsystem 77"
Cohesion: 0.28
Nodes (3): CreateItemInput, Item, UpdateItemInput

### Community 78 - "Purchasing & Procurement"
Cohesion: 0.25
Nodes (4): CreatePRInput, CreatePRItemInput, UpdatePRInput, PurchaseRequisition

### Community 79 - "IAM & Platform Infrastructure"
Cohesion: 0.25
Nodes (6): __dirname, nextConfig, projectRoot, ref_next_env, ref_node_path, ref_node_url

### Community 80 - "Sales & Customer Distribution"
Cohesion: 0.32
Nodes (5): DBExecutor, SalesQuotation, GenerateDocumentNumberWithBranch(), GenerateNextDocNumber(), pgx.Row

### Community 85 - "Purchasing & Procurement"
Cohesion: 0.25
Nodes (4): InfraHealthResponse, PostgresHealth, S3Health, ValkeyHealth

### Community 87 - "Domain Subsystem 87"
Cohesion: 0.38
Nodes (5): apps_web_app_globals, metadata, RootLayout(), Providers(), next-themes

### Community 89 - "Purchasing & Procurement"
Cohesion: 0.29
Nodes (3): bufio.ReadWriter, net.Conn, statusResponseWriter

### Community 90 - "Inventory & Warehouse Movements"
Cohesion: 0.29
Nodes (3): CreateWarehouseInput, Warehouse, Warehouse

### Community 91 - "Manufacturing Execution & Job Costing"
Cohesion: 0.29
Nodes (3): CreateBOMRoutingInput, BOMRouting, BOMRouting

### Community 93 - "Domain Subsystem 93"
Cohesion: 0.33
Nodes (3): CreatePickingSlipInput, PickingSlip, PickingSlip

### Community 98 - "Purchasing & Procurement"
Cohesion: 0.33
Nodes (3): CreateRFQInput, CreateRFQItemInput, RFQ

### Community 100 - "Frontend Shared UI & Document Engine"
Cohesion: 0.40
Nodes (5): scripts, build, dev, lint, start

### Community 107 - "Purchasing & Procurement"
Cohesion: 0.40
Nodes (3): SubmitVendorBidInput, VendorBid, VendorBid

### Community 108 - "Domain Subsystem 108"
Cohesion: 0.50
Nodes (3): TestBiometricDeviceValidation(), BiometricLogItemInput, DeviceSyncLogInput

### Community 109 - "Domain Subsystem 109"
Cohesion: 0.50
Nodes (3): extends, ignorePatterns, next/core-web-vitals

## Knowledge Gaps
- **238 isolated node(s):** `extends`, `next/core-web-vitals`, `ignorePatterns`, `loginSchema`, `LoginFormValues` (+233 more)
  These have ≤1 connection - possible missing edges. (Counts symbols only; 374 node(s) total have ≤1 connection when file, concept and rationale nodes are included.)
- **39 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `main()` connect `Sales & Customer Distribution` to `Domain Subsystem 4`, `Purchasing & Procurement`, `Database Schema & Persistence`, `HRIS & Indonesian Statutory Payroll`, `IAM & Platform Infrastructure`, `Sales & Customer Distribution`, `IAM & Platform Infrastructure`, `Domain Subsystem 17`, `Domain Subsystem 18`, `Sales & Customer Distribution`, `Accounting & Financial Ledger`, `Inventory & Warehouse Movements`, `Sales & Customer Distribution`, `Purchasing & Procurement`, `Sales & Customer Distribution`, `Quality Assurance & ISO Control`, `Purchasing & Procurement`, `Frontend Shared UI & Document Engine`, `IAM & Platform Infrastructure`, `Sales & Customer Distribution`, `Purchasing & Procurement`, `Domain Subsystem 46`, `Sales & Customer Distribution`, `Purchasing & Procurement`, `Purchasing & Procurement`, `Purchasing & Procurement`, `Domain Subsystem 70`?**
  _High betweenness centrality (0.048) - this node is a cross-community bridge._
- **Why does `react` connect `Purchasing & Procurement` to `Frontend Shared UI & Document Engine`, `Frontend Shared UI & Document Engine`, `Purchasing & Procurement`, `HRIS & Indonesian Statutory Payroll`, `Frontend Shared UI & Document Engine`, `Purchasing & Procurement`, `Manufacturing Execution & Job Costing`, `Purchasing & Procurement`, `Sales & Customer Distribution`, `Accounting & Financial Ledger`, `IAM & Platform Infrastructure`, `Domain Subsystem 30`, `Frontend Shared UI & Document Engine`, `Purchasing & Procurement`, `Purchasing & Procurement`, `Frontend Shared UI & Document Engine`, `IAM & Platform Infrastructure`, `Domain Subsystem 87`, `Purchasing & Procurement`?**
  _High betweenness centrality (0.034) - this node is a cross-community bridge._
- **Why does `Success()` connect `HRIS & Indonesian Statutory Payroll` to `Accounting & Financial Ledger`, `Domain Subsystem 66`, `Domain Subsystem 3`, `IAM & Platform Infrastructure`, `IAM & Platform Infrastructure`, `Sales & Customer Distribution`, `Sales & Customer Distribution`, `Sales & Customer Distribution`, `Quality Assurance & ISO Control`, `Domain Subsystem 53`, `Treasury & Bank Reconciliation`, `Purchasing & Procurement`, `Sales & Customer Distribution`, `Domain Subsystem 94`?**
  _High betweenness centrality (0.033) - this node is a cross-community bridge._
- **What connects `extends`, `next/core-web-vitals`, `ignorePatterns` to the rest of the system?**
  _238 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Frontend Shared UI & Document Engine` be split into smaller, more focused modules?**
  _Cohesion score 0.05176342550079924 - nodes in this community are weakly interconnected._
- **Should `Frontend Shared UI & Document Engine` be split into smaller, more focused modules?**
  _Cohesion score 0.07192807192807193 - nodes in this community are weakly interconnected._
- **Should `Purchasing & Procurement` be split into smaller, more focused modules?**
  _Cohesion score 0.07811921510551648 - nodes in this community are weakly interconnected._