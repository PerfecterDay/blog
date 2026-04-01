docker pull mcr.microsoft.com/mssql/server:2022-latest

docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=Aa123456789#" \
-p 1433:1433 --name sql1 --hostname sql1 \
-d \
mcr.microsoft.com/mssql/server:2022-latest

docker exec -it sql1 "bash"

/opt/mssql-tools18/bin/sqlcmd -No -S localhost -U SA -P "Aa123456789#" //连接数据库

SELECT name FROM sys.databases; go //查看当前有哪些数据库
create database etradepro; go //创建数据库

select name from sysobjects where type = 'P'; go //查看存储过程


/opt/mssql-tools18/bin/sqlcmd -No -S localhost -U SA -P "Aa123456789#" -d etradepro -Q "create PROCEDURE [dbo].[API_CLNTSETTING_LIST](
	@ClntCode	varchar(10),
	@SettingName	varchar(50)=''  
)
AS
SET NOCOUNT ON 
if  @SettingName ='NOSMS'
begin 
	Select	ClntCode = @ClntCode ,SettingName = @SettingName ,SettingValue ='NOSMS value'  
end
else
begin
   Select	ClntCode = @ClntCode ,SettingName = @SettingName ,SettingValue = 'xxxxxxx'  	
end"


/opt/mssql-tools18/bin/sqlcmd -No -S localhost -U SA -P "Aa123456789#" -d etradepro -Q 'drop PROCEDURE [dbo].[API_CLNTSETTING_LIST]'


/opt/mssql-tools18/bin/sqlcmd -No -S localhost -U SA -P "Aa123456789#" -d ClntAuth -Q "CREATE PROCEDURE [dbo].[GJ_CLNTTRADETOKENCHECK](
  @clntcode varchar(20),
  @TradeToken varchar(100),
  @APINO	varchar(50) 
)
AS  
Set NoCount On
 
	Declare @Status Varchar(10), @bgStatus Varchar(10), @ErrMsg nvarchar(100)
	if ( @clntcode = '0' )  
	begin
		select Result = 0  , Errorcode = 0    --  令牌正確			
	end
	else
	begin		
		select Result = 1  , Errorcode = 82   		
	end
 
	Return
"

/opt/mssql-tools18/bin/sqlcmd -No -S localhost -U SA -P "Aa123456789#" -d ClntAuth -Q 'drop PROCEDURE [dbo].[GJ_CLNTTRADETOKENCHECK]'

QUIT


/opt/mssql-tools18/bin/sqlcmd -No -S localhost -U SA -P "Aa123456789#" -d etradepro -Q "create  PROCEDURE  [dbo].[API_CLNTNAMELIST] 
(
	@ClntCode	Varchar(20),
	@Opuser		Varchar(20) ='Internet'
)
AS
SET NOCOUNT ON
 
if ( @Opuser = 'Internet')
Begin
	select ClntName='6838LastName_Eng 6838FirstName_Eng'
End
else
begin
	RaisError ('Invalid Opuser', 18, 1);  
	Return 
end"