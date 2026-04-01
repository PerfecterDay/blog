# Mixin
{docsify-updated}

```
@Data
public class CommonDebitCreditRequest {
    private String functionId; //功能号
    private String operWayExt; //委托方式 04
    private String stationAddr; //操作站点 MAC地址
    private String terminalInfo; //终端信息
    private String cifAccount; //一户通账号
    private String withdrawFundAccount; //出金资金账号
    private String depositFundAccount; //入金资金账号
    private String businessType; //业务种类：1-北向通入金，2-北向通出金， 3-南向通入金，4-南向通出金 1-4
    private String ccy; //币种代码：CNY-人民币，HKD-港币，USD-美元  CNY
    private String transferAmount; //转账金额
    private String customerName; //客户姓名
    private String customerIdType; //客户证件类型
    private String customerId; //客户证件号码
    private String referenceNo; //客户证件号码
    private String remark; //备注
    private String trustCode; //互信码


    public interface MyMixin {

        @JsonProperty("FUNCTION_ID")
        public String getFunctionId();

        @JsonProperty("OPERWAY_EXT")
        public String getOperWayExt();

        @JsonProperty("STATION_ADDR")
        public String getStationAddr();

        @JsonProperty("TERMINALINFO")
        public String getTerminalInfo();

        @JsonProperty("CIF_ACCOUNT")
        public String getCifAccount();

        @JsonProperty("WITHDRAWAL_FUND_ACCOUNT")
        public String getWithdrawFundAccount();

        @JsonProperty("DEPOSIT_FUND_ACCOUNT")
        public String getDepositFundAccount();

        @JsonProperty("BUSINESS_TYPE")
        public String getBusinessType();

        @JsonProperty("CCY")
        public String getCcy();

        @JsonProperty("TRANSFER_AMOUNT")
        public String getTransferAmount();

        @JsonProperty("CUSTOMER_NAME")
        public String getCustomerName();

        @JsonProperty("CUSTOMER_IDTYPE")
        public String getCustomerIdType();

        @JsonProperty("CUSTOMER_ID")
        public String getCustomerId();

        @JsonProperty("REFERENCE_NO")
        public String getReferenceNo();


        @JsonProperty("REMARK")
        public String getRemark();

        @JsonProperty("TRUST_CODE")
        public String getTrustCode();
    }
}

String s = mapper.addMixIn(CrossBorderDebitCreditLimitCheckRequest.class, CommonDebitCreditRequest.MyMixin.class).writeValueAsString(request);
```