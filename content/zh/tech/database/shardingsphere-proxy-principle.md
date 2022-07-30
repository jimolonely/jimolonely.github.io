---
title: "ShardingSphere proxy原理"
date: 2022-05-21T16:42:13+08:00
tags: [ "shardingsphere" ]
featured_image: ""
description: ""
---

代理分为前端和后端。

类 `MetaDataContextsBuilder`会根据配置文件构建数据的上下文，其中包括 前端和后端数据库类型。


## 调用流程

FrontendChannelInboundHandler

```java
@Override
public void channelRead(final ChannelHandlerContext context, final Object message) {
    if (!authenticated) {
        authenticated = authenticate(context, (ByteBuf) message);
        return;
    }
    ProxyStateContext.execute(context, message, databaseProtocolFrontendEngine, connectionSession);
}
```

然后采用JDBC代理
JDBCOKProxyState，这里使用了一个异步线程去处理。

```java
@Override
public void execute(final ChannelHandlerContext context, final Object message, final DatabaseProtocolFrontendEngine databaseProtocolFrontendEngine, final ConnectionSession connectionSession) {
    CommandExecutorTask commandExecutorTask = new CommandExecutorTask(databaseProtocolFrontendEngine, connectionSession, context, message);
    ExecutorService executorService = determineSuitableExecutorService(context, databaseProtocolFrontendEngine, connectionSession);
    executorService.execute(commandExecutorTask);
}
```

CommandExecutorTask

```java
private boolean executeCommand(final ChannelHandlerContext context, final PacketPayload payload) throws SQLException {
    CommandExecuteEngine commandExecuteEngine = databaseProtocolFrontendEngine.getCommandExecuteEngine();
    CommandPacketType type = commandExecuteEngine.getCommandPacketType(payload);
    CommandPacket commandPacket = commandExecuteEngine.getCommandPacket(payload, type, connectionSession);
    CommandExecutor commandExecutor = commandExecuteEngine.getCommandExecutor(type, commandPacket, connectionSession);
    try {
        Collection<DatabasePacket<?>> responsePackets = commandExecutor.execute();
        if (responsePackets.isEmpty()) {
            return false;
        }
        responsePackets.forEach(context::write);
        if (commandExecutor instanceof QueryCommandExecutor) {
            commandExecuteEngine.writeQueryData(context, connectionSession.getBackendConnection(), (QueryCommandExecutor) commandExecutor, responsePackets.size());
        }
        return true;
    } catch (final SQLException ex) {
        databaseProtocolFrontendEngine.handleException(connectionSession);
        throw ex;
    } finally {
        commandExecutor.close();
    }
}
```

首先，前端数据库有以下几种实现：

![image](https://user-images.githubusercontent.com/17684996/169645143-057d5f1a-9e2f-424d-bd21-707b6e1695f2.png)

执行引擎有几个数据库实现

![image](https://user-images.githubusercontent.com/17684996/169645220-14cc497e-6335-4a1c-9d51-49a62a761190.png)

我们这里看MySQL的，根据其协议内容，有下面的一些命令：

```java
public static CommandExecutor newInstance(final MySQLCommandPacketType commandPacketType, final CommandPacket commandPacket, final ConnectionSession connectionSession) throws SQLException {
    log.debug("Execute packet type: {}, value: {}", commandPacketType, commandPacket);
    switch (commandPacketType) {
        case COM_QUIT:
            return new MySQLComQuitExecutor();
        case COM_INIT_DB:
            return new MySQLComInitDbExecutor((MySQLComInitDbPacket) commandPacket, connectionSession);
        case COM_FIELD_LIST:
            return new MySQLComFieldListPacketExecutor((MySQLComFieldListPacket) commandPacket, connectionSession);
        case COM_QUERY:
            return new MySQLComQueryPacketExecutor((MySQLComQueryPacket) commandPacket, connectionSession);
        case COM_PING:
            return new MySQLComPingExecutor();
        case COM_STMT_PREPARE:
            return new MySQLComStmtPrepareExecutor((MySQLComStmtPreparePacket) commandPacket, connectionSession);
        case COM_STMT_EXECUTE:
            return new MySQLComStmtExecuteExecutor((MySQLComStmtExecutePacket) commandPacket, connectionSession);
        case COM_STMT_RESET:
            return new MySQLComStmtResetExecutor((MySQLComStmtResetPacket) commandPacket);
        case COM_STMT_CLOSE:
            return new MySQLComStmtCloseExecutor((MySQLComStmtClosePacket) commandPacket, connectionSession.getConnectionId());
        case COM_SET_OPTION:
            return new MySQLComSetOptionExecutor((MySQLComSetOptionPacket) commandPacket, connectionSession);
        default:
            return new MySQLUnsupportedCommandExecutor(commandPacketType);
    }
}
```

在SS里，解析完协议后，每种命令都给了一种执行实现，我不确定用了哪种命令，就打断点看了一下：

![fa43bf8da6c0fcfb89ed99b56f89f4b](https://user-images.githubusercontent.com/17684996/169645957-7aae0036-668c-4dc6-b43f-89144ff69cc5.png)

![03dfbd5aebdb7459a49e9aab57544d8](https://user-images.githubusercontent.com/17684996/169645967-11d7f477-ff96-455d-a8c1-21a995658db4.png)

![956d4f17f5b7c59900697d083259122](https://user-images.githubusercontent.com/17684996/169645970-73c61bc9-ebf8-49f6-bc19-2c206956f7fb.png)

发现都是 `MySQLComQueryPacketExecutor`, 在这个实现里，主要关注其 `execute`方法

```java
@Override
public Collection<DatabasePacket<?>> execute() throws SQLException {
    ResponseHeader responseHeader = textProtocolBackendHandler.execute();
    if (responseHeader instanceof QueryResponseHeader) {
        return processQuery((QueryResponseHeader) responseHeader);
    }
    responseType = ResponseType.UPDATE;
    return responseHeader instanceof UpdateResponseHeader ? processUpdate((UpdateResponseHeader) responseHeader) : processClientEncoding((ClientEncodingResponseHeader) responseHeader);
}
```

在这里, `textProtocolBackendHandler.execute()`又有很多种实现，从上图可以看到是 `SchemaAssignedDatabaseBackendHandler`：

```java
private DatabaseCommunicationEngine<?> databaseCommunicationEngine;

@Override
public ResponseHeader execute() throws SQLException {
    prepareDatabaseCommunicationEngine();
    return (ResponseHeader) databaseCommunicationEngine.execute();
}
```
这里使用的 `databaseCommunicationEngine`实现是 `JDBCDatabaseCommunicationEngine`:


```java
public ResponseHeader execute() {
    LogicSQL logicSQL = getLogicSQL();
    ExecutionContext executionContext = getKernelProcessor().generateExecutionContext(
            logicSQL, getDatabase(), ProxyContext.getInstance().getContextManager().getMetaDataContexts().getProps());
    // TODO move federation route logic to binder
    SQLStatementContext<?> sqlStatementContext = logicSQL.getSqlStatementContext();
    String defaultDatabaseName = backendConnection.getConnectionSession().getDatabaseName();
    if (executionContext.getRouteContext().isFederated() || (sqlStatementContext instanceof SelectStatementContext
            && SystemSchemaUtil.containsSystemSchema(sqlStatementContext.getDatabaseType(), sqlStatementContext.getTablesContext().getSchemaNames(), defaultDatabaseName))) {
        MetaDataContexts metaDataContexts = ProxyContext.getInstance().getContextManager().getMetaDataContexts();
        ResultSet resultSet = doExecuteFederation(logicSQL, metaDataContexts);
        return processExecuteFederation(resultSet, metaDataContexts);
    }
    if (executionContext.getExecutionUnits().isEmpty()) {
        return new UpdateResponseHeader(executionContext.getSqlStatementContext().getSqlStatement());
    }
    proxySQLExecutor.checkExecutePrerequisites(executionContext);
    checkLockedDatabase(executionContext);
    List result = proxySQLExecutor.execute(executionContext);
    refreshMetaData(executionContext);
    Object executeResultSample = result.iterator().next();
    return executeResultSample instanceof QueryResult
            ? processExecuteQuery(executionContext, result, (QueryResult) executeResultSample)
            : processExecuteUpdate(executionContext, result);
}
```

我们不是这里的联邦查询，所以真正的执行在 `proxySQLExecutor.checkExecutePrerequisites(executionContext);`

在 ProxySQLExecutor 里：

```java
public List<ExecuteResult> execute(final ExecutionContext executionContext) throws SQLException {
    String databaseName = backendConnection.getConnectionSession().getDatabaseName();
    Collection<ShardingSphereRule> rules = ProxyContext.getInstance().getContextManager().getMetaDataContexts().getDatabaseMetaData(databaseName).getRuleMetaData().getRules();
    int maxConnectionsSizePerQuery = ProxyContext.getInstance().getContextManager().getMetaDataContexts().getProps().<Integer>getValue(ConfigurationPropertyKey.MAX_CONNECTIONS_SIZE_PER_QUERY);
    boolean isReturnGeneratedKeys = executionContext.getSqlStatementContext().getSqlStatement() instanceof MySQLInsertStatement;
    return execute(executionContext, rules, maxConnectionsSizePerQuery, isReturnGeneratedKeys);
}

private List<ExecuteResult> execute(final ExecutionContext executionContext, final Collection<ShardingSphereRule> rules,
final int maxConnectionsSizePerQuery, final boolean isReturnGeneratedKeys) throws SQLException {
return hasRawExecutionRule(rules) ? rawExecute(executionContext, rules, maxConnectionsSizePerQuery)
: useDriverToExecute(executionContext, rules, maxConnectionsSizePerQuery, isReturnGeneratedKeys, SQLExecutorExceptionHandler.isExceptionThrown());
}
```

这里 `backendConnection`的实现是 `JDBCBackendConnection`. 采用的是 `useDriverToExecute`

```java
private List<ExecuteResult> useDriverToExecute(final ExecutionContext executionContext, final Collection<ShardingSphereRule> rules,
                                               final int maxConnectionsSizePerQuery, final boolean isReturnGeneratedKeys, final boolean isExceptionThrown) throws SQLException {
    JDBCBackendStatement statementManager = (JDBCBackendStatement) backendConnection.getConnectionSession().getStatementManager();
    DriverExecutionPrepareEngine<JDBCExecutionUnit, Connection> prepareEngine = new DriverExecutionPrepareEngine<>(
            type, maxConnectionsSizePerQuery, backendConnection, statementManager, new StatementOption(isReturnGeneratedKeys), rules);
    ExecutionGroupContext<JDBCExecutionUnit> executionGroupContext;
    try {
        executionGroupContext = prepareEngine.prepare(executionContext.getRouteContext(), executionContext.getExecutionUnits());
    } catch (final SQLException ex) {
        return getSaneExecuteResults(executionContext, ex);
    }
    executionGroupContext.setDatabaseName(backendConnection.getConnectionSession().getDatabaseName());
    executionGroupContext.setGrantee(backendConnection.getConnectionSession().getGrantee());
    return jdbcExecutor.execute(executionContext.getLogicSQL(), executionGroupContext, isReturnGeneratedKeys, isExceptionThrown);
}
```

使用的 `ProxyJDBCExecutor`
```java
jdbcExecutor.execute(executionContext.getLogicSQL(), executionGroupContext, isReturnGeneratedKeys, isExceptionThrown);
```
这里面又调用了一个 `JDBCExecutor`来最终执行，这里的关键点是 `ProxyJDBCExecutorCallbackFactory.newInstance(type, databaseType, context.getSqlStatement(), databaseCommunicationEngine, isReturnGeneratedKeys, isExceptionThrown, false)`会创建一个回调。
```java
public List<ExecuteResult> execute(final LogicSQL logicSQL, final ExecutionGroupContext<JDBCExecutionUnit> executionGroupContext,
                                   final boolean isReturnGeneratedKeys, final boolean isExceptionThrown) throws SQLException {
    try {
        MetaDataContexts metaDataContexts = ProxyContext.getInstance().getContextManager().getMetaDataContexts();
        DatabaseType databaseType = metaDataContexts.getDatabaseMetaData(connectionSession.getDatabaseName()).getResource().getDatabaseType();
        ExecuteProcessEngine.initialize(logicSQL, executionGroupContext, metaDataContexts.getProps());
        SQLStatementContext<?> context = logicSQL.getSqlStatementContext();
        List<ExecuteResult> result = jdbcExecutor.execute(executionGroupContext,
                ProxyJDBCExecutorCallbackFactory.newInstance(type, databaseType, context.getSqlStatement(), databaseCommunicationEngine, isReturnGeneratedKeys, isExceptionThrown, true),
                ProxyJDBCExecutorCallbackFactory.newInstance(type, databaseType, context.getSqlStatement(), databaseCommunicationEngine, isReturnGeneratedKeys, isExceptionThrown, false));
        ExecuteProcessEngine.finish(executionGroupContext.getExecutionID());
        return result;
    } finally {
        ExecuteProcessEngine.clean();
    }
}
```


真正的执行会到 `ProxyStatementExecutorCallback`回调里用 JDBC原生Statement去执行。

```java
public final class ProxyStatementExecutorCallback extends ProxyJDBCExecutorCallback {
    
    @Override
    protected boolean execute(final String sql, final Statement statement, final boolean isReturnGeneratedKeys) throws SQLException {
        return statement.execute(sql, isReturnGeneratedKeys ? Statement.RETURN_GENERATED_KEYS : Statement.NO_GENERATED_KEYS);
    }
}
```








