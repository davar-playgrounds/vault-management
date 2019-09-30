vault-management
================

This role will be used to create a single Hashicorp Vault instance with consul as the backend. It will use OpenSSL to generate a self-signed SSL certificate to use for communication with Vault over HTTPS and create a cron job to backup consul snapshots to AWS S3 every hour. The role will also create a vault and consul user group and user respectively to be used to manage the vault and consul services. Lastly, the role will configure the firewalld configuration such that a user can access the User Interface (UI) for Vault for a more manageable interface than just the command line. Optionally, this role may also be used to create a new Vault instance restored from an existing consul snapshot in S3 instead of creating an empty Vault. For details on how to perform this action, please see the Vault Backup Restoration section below.

Requirements
------------

* The specific version of Vault or Consul can be specified in the defaults/main.yml file, the releases can be retrieved from releases.hashicorp.com.
* AWS EC2 User created with inline policy or AWS IAM role
* Vault instance with IAM instance profile with the following permissions:
    * read/write permissions to AWS S3 bucket `consul_backup_s3_bucket`
    * encrypt/decrypt/describe permissions to AWS KMS keys

Role Variables
--------------

| variable name                       | description                                                                                                                                                                  |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| consul_address                      | address that consul will be listening on                                                                                                                                     |
| consul_download_url                 | web uri for retrieving the desired release from hashicorp for consul                                                                                                         |
| consul_account_name                 | name of the consul user and group                                                                                                                                            |
| consul_config_path                  | location for the files that will be used to configure consul                                                                                                                 |
| consul_backup_s3_bucket             | s3 bucket name to store consul backups, if left empty, vault server will not be configured to back itself up to remote AWS S3 bucket                                         |
| consul_backup_directory             | local directory on vault server to store consul backups                                                                                                                      |
| consul_snapshot_s3_restoration_key  | aws s3 key to remote snapshot file to automatically restore new vault from, default left empty meaning no restoration will be performed and role will create new empty vault |
| consul_snapshot_s3_restore_in_place | restore vault from remote consul snapshot in s3 in place when set to true                                                                                                    |
| vault_tcp_address                   | address that vault will listen on                                                                                                                                            |
| disable_vault_tls                   | when value is set to 1 tls is disabled and Vault must be accessed via http                                                                                                   |
| vault_config_path                   | location for the files that will be used to configure vault                                                                                                                  |
| vault_download_url                  | web uri for retrieving the desired release from hashicorp for vault                                                                                                          |
| vault_account_name                  | name of the vault user and group                                                                                                                                             |
| aws_kms_key_region                  | AWS region with KMS key to unseal Vault                                                                                                                                      |
| aws_kms_key_id                      | AWS KMS key id to auto-unseal Vault                                                                                                                                          |
| vault_create_users                  | boolean value which will create users defined in defaults/main.yml                                                                                                           |
| vault_admins                        | dictionary of vault admin accounts to create by default                                                                                                                      |
| vault_engines_enable_ssh            | only enable ssh secrets engines when set to true                                                                                                                             |
| vault_engines_ssh                   | dictionary of names for ssh secrets engines to enable, please see comments in defaults/main.yml for details                                                                  |
| vault_engines_enable_aws            | only enable aws secrets engines when set to true                                                                                                                             |
| vault_aws_bound_iam_role            | AWS IAM role to permit access to all vault secrets                                                                                                                           |

Additional Information
----------------------

### Vault Backups

This role configures Vault automatically with a cron job that backups up a consul snapshot to AWS S3 bucket `consul_backup_s3_bucket` every hour. It stores these snapshot within the bucket in directory `/<VAULT INSTANCE ID>/<DATE>/consul-snapshot-<TIMESTAMP>-<VAULT INSTANCE ID>.bin`.

To manually backup the Vault server on demand, please run the following command on the Vault server:
```
bash /etc/consul.d/backup-consul.sh
```

### AWS KMS Auto-Unsealing Vault

In order for Vault to auto-unseal on startup, it must make use of the `aws_kms_key_id` default variable to point to a pre-created AWS KMS key created for the purpose of unsealing Vault.

To assist in the absence of using Terraform to handle KMS key configuration in AWS, the manual aws cli steps to configure a KMS key have been provided below:

```
cat > REDACTED-vault-kms-unseal-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VaultKMSUnseal",
            "Effect": "Allow",
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:DescribeKey"
            ],
            "Resource": "*"
        }
    ]
}
EOF
aws iam create-policy --policy-name REDACTED-vault-kms-unseal-policy --policy-document file://REDACTED-vault-kms-unseal-policy.json
aws iam attach-role-policy --policy-arn arn:aws-us-gov:iam::REDACTED:policy/REDACTED-vault-kms-unseal-policy --role-name REDACTED-vault
KMS_KEY_ID=$(aws kms create-key --description "Vault unseal key" --output text --query "KeyMetadata.KeyId")
aws kms create-alias --alias-name alias/vault-kms-unseal-key --target-key-id $KMS_KEY_ID
```

### New Vault Backup Restoration

This role offers the ability to create a new Vault instance restored from an existing consul snapshot stored in AWS S3 bucket `consul_backup_s3_bucket` as opposed to creating a new clean Vault. To perform this action, please follow the instructions below:

1. Copy the `Key` vault of the desired consul snapshot to restore from stored in AWS S3 bucket `consul_backup_s3_bucket`
2. Execute the playbook that includes the `vault-management` role with the `consul_snapshot_s3_restoration_key` extra variable set to the copied `Key` from step 1 above.
   
    ```
    ansible-playbook [OPTIONS] <PLAYBOOK NAME> -e "consul_snapshot_s3_restoration_key=<KEY OF CONSUL SNAPSHOT IN S3>"
    ```
    
    Example:
    
    ```
    ansible-playbook -i localhost, -c local /etc/ansible/playbooks/vault-create.yml -e "consul_snapshot_s3_restoration_key=REDACTED/2019.08.06/consul-snapshot-2019.08.06-18:11:19-REDACTED.bin"
    ```
    
### Vault Backup Restoration In Place

This role offers in addition to the functionality to create a new Vault instance from an existing consul snapshot the ability to live restore an active Vault in place from an existing consul snapshot stored in stored in AWS S3 bucket `consul_backup_s3_bucket`. To perform this action, please follow the instructions below:

1. Copy the `Key` vault of the desired consul snapshot to restore from stored in AWS S3 bucket `consul_backup_s3_bucket`
2. Execute the playbook that includes the `vault-management` role with the `consul_snapshot_s3_restore_in_place` extra variable set to `true` and the `consul_snapshot_s3_restoration_key` extra variable set to the copied `Key` from step 1 above.

    ```
    ansible-playbook [OPTIONS] <PLAYBOOK NAME> -e "consul_snapshot_s3_restore_in_place=true consul_snapshot_s3_restoration_key=<KEY OF CONSUL SNAPSHOT IN S3>"
    ```
    
    Example:
    
    ```
    ansible-playbook -i localhost, -c local /etc/ansible/playbooks/vault-create.yml -e "consul_snapshot_s3_restore_in_place=true consul_snapshot_s3_restoration_key=REDACTED/2019.08.06/consul-snapshot-2019.08.06-18:11:19-REDACTED.bin"
    ```

### AWS EC2 Authentication

In order to have Vault properly query AWS, the following AWS IAM User should be created and configured with the following IAM inline policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "iam:GetInstanceProfile",
                "iam:GetUser",
                "iam:GetRole"
            ],
            "Resource": "*"
        }
    ]
}
```

This comes into play when enabling aws auth methods during the configuration phase. Eventually this should be replaced with an IAM role that can be applied to the Vault server.

### BASH Environment Variables

At a minmum `VAULT_ADDR` must be defined, when `vault login` is not issued, `VAULT_TOKEN` should also be set; additional environment variables can be found on the Hashicorp site: https://www.vaultproject.io/docs/commands/#environment-variables

Example Playbook
----------------

```
    - hosts: localhost
      roles:
         - vault-management
```

Author Information
------------------

* Alijohn Ghassemlouei (aghassemlouei)
* Devon Thyne (devon-thyne)
