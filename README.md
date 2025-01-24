USE [master];
GO

-- Specify the login name
DECLARE @LoginName NVARCHAR(128) = 'YourLoginName';

-- Revoke server-level permissions
DECLARE @ServerPermission NVARCHAR(128);
DECLARE ServerPermissions CURSOR FOR
SELECT sp.permission_name
FROM sys.server_permissions sp
JOIN sys.server_principals spr ON sp.grantee_principal_id = spr.principal_id
WHERE spr.name = @LoginName;

OPEN ServerPermissions;
FETCH NEXT FROM ServerPermissions INTO @ServerPermission;

WHILE @@FETCH_STATUS = 0
BEGIN
    DECLARE @Sql NVARCHAR(MAX) = N'REVOKE ' + @ServerPermission + ' TO [' + @LoginName + '];';
    EXEC sp_executesql @Sql;
    FETCH NEXT FROM ServerPermissions INTO @ServerPermission;
END

CLOSE ServerPermissions;
DEALLOCATE ServerPermissions;

-- Revoke database-level permissions for all databases
DECLARE @DatabaseName NVARCHAR(128);
DECLARE Databases CURSOR FOR
SELECT name
FROM sys.databases
WHERE state_desc = 'ONLINE';

OPEN Databases;
FETCH NEXT FROM Databases INTO @DatabaseName;

WHILE @@FETCH_STATUS = 0
BEGIN
    DECLARE @Sql NVARCHAR(MAX) = '';
    SET @Sql = 'USE [' + @DatabaseName + ']; ' +
               'DECLARE @DbPermission NVARCHAR(128); ' +
               'DECLARE DbPermissions CURSOR FOR ' +
               'SELECT dp.permission_name ' +
               'FROM sys.database_permissions dp ' +
               'JOIN sys.database_principals dpr ON dp.grantee_principal_id = dpr.principal_id ' +
               'WHERE dpr.name = ''' + @LoginName + '''; ' +

               'OPEN DbPermissions; ' +
               'FETCH NEXT FROM DbPermissions INTO @DbPermission; ' +

               'WHILE @@FETCH_STATUS = 0 ' +
               'BEGIN ' +
               '   EXEC(''REVOKE '' + @DbPermission + '' TO [' + @LoginName + '];''); ' +
               '   FETCH NEXT FROM DbPermissions INTO @DbPermission; ' +
               'END ' +

               'CLOSE DbPermissions; ' +
               'DEALLOCATE DbPermissions;';

    EXEC sp_executesql @Sql;
    FETCH NEXT FROM Databases INTO @DatabaseName;
END

CLOSE Databases;
DEALLOCATE Databases;

PRINT 'All permissions revoked for login: ' + @LoginName;
