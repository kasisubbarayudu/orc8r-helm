# orc8r-helm

> [!NOTE]
> Replace the above values with yours.
>
> Set `CROSS_ACCOUNT_ACCESS` to "True" to enable cross-account access support.
 
> [!IMPORTANT]
> If the `CROSS_ACCOUNT_ACCESS` is set to True:
>
> :heavy_check_mark: [ Mandatory ]: Set `"IAM_ROLE_ARN"` to the ARN of the IAM role in the target account.
>
> :heavy_check_mark: [ Mandatory ]: Set `"SOURCE_PROFILE"` to the AWS CLI named profile containing the source account's credentials.

