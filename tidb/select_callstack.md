* optimizer callstack:
  ```
  Compiler::Compile -> Preprocess -> preprocessor visit //actions such as fill the dbname of table, preprocessor::handleTableName
                    |_ Optimize -> build planBuilder
                    |           |_ planBuilder::build -> planBuilder::buildSelect -> planBuilder::buildResultSetNode ... -> planBuilder::buildDataSource -> getPossibleAccessPaths
                    |           |                                                 |                                                                      |_ build DataSource
                    |           |                                                 |_ planBuilder::unfoldWildStar
                    |           |                                                 |_ planBuilder::buildSelection -> splitWhere ----------------------
                    |           |                                                 |                              |_ SplitCNFItems                   |-> logical reduction of WHERE recursively
                    |           |                                                 |                              |_ EvalBool //evaluate const expr---
                    |           |                                                 |                              |_ SetChildren //LogicalSelection as the parent node
                    |           |                                                 |_ planBuilder::buildProjection //LogicalProjection as the parent node
                    |           |                                                 |_ return LogicalPlan
                    |           |_ doOptimize -> logicalOptimize //apply rules, columnPruner, projectionEliminater, and ppdSolver usually, in recursive style
                    |                         |_ physicalOptimize -> ::deriveStats
                    |                         |                   |_ ::findBestTask -> ::exhaustPhysicalPlans //multiple implementations for agg, join, sort etc
                    |                         |                   |                 |_ foreach pp, combine it with best children tasks(::attach2Task), compute the best task with least cost
                    |                         |                   |_ ::ResolveIndices //set Column::Index in the plan tree, according to Schema
                    |                         |_ eliminatePhysicalProjection -> canProjectionBeEliminatedStrict //Schema() exactly same as child
                    |_ build ExecStmt

  DataSource::PredicatePushDown -> ExpressionsToPB
  LogicalSelection::PredicatePushDown would return child, i.e, DataSource, hence remove LogicalSelection node from the tree in addSelection()

  DataSource::findBestTask -> foreach accessPath DataSource::convertToTableScan -> PhysicalTableScan
                                                                                |_ copTask
                                                                                |_ PhysicalTableScan::addPushedDownSelection -> PhysicalSelection
                                                                                |                                            |_ SetChildren
                                                                                |                                            |_ copTask.tablePlan = sel
                                                                                |_ finishCopTask -> rootTask
                                                                                                 |_ PhysicalTableReader

  interface inheritance relationship:
  Plan -> LogicalPlan
       |_ PhysicalPlan

  struct inheretance:
  basePlan -> baseLogicalPlan
           |_ basePhysicalPlan

  Plan::Schema() represents the output fields of this operator
  ```

* XXX: join table order? combinations?

* executor callstack:
  ```
  session::executeStatement -> runStmt -> ExecStmt::Exec -> ExecStmt::buildExecutor -> newExecutorBuilder
                                                         |                          |_ executorBuilder::build -> executorBuilder::buildTableReader
                                                         |_ TableReaderExecutor::Open -> TableReaderExecutor::buildResp -> distsql.Select -> Client::Send
                                                         |                                                              |                 |_ selectResult
                                                         |                                                              |_ selectResult::Fetch -> go selectResult::fetch //async, using goroutine pool, write results into chan
                                                         |_ recordSet

  executorBuilder::buildTableReader -> buildNoRangeTableReader -> executorBuilder::constructDAGReq -> constructDistExec -> foreach PhysicalPlan ::ToPB
                                    |                          |_ TableReaderExecutor
                                    |_ TableReaderExecutor::ranges = PhysicalTableScan::Ranges

  clientConn::handleQuery -> session::Execute
                          |_ clientConn::writeResultSet -> clientConn::writeChunks -> tidbResultSet::Next -> ... -> selectResult::Next //read chan
                                                                                   |_ clientConn::writePacket
  ```