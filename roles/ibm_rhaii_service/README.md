# IBM RHAII Service Role

This Ansible role automates the deployment and management of IBM RHAII service instances on IBM Cloud. It provides
complete lifecycle management including provisioning, configuration, and destruction of RHAII resources with
enterprise-grade error handling and logging.

## Features

- **Complete Lifecycle Management**: Provision and destroy RHAII service instances
- **Idempotent Operations**: Safe to run multiple times with consistent results
- **Enterprise Error Handling**: Comprehensive retry logic and descriptive error messages
- **Terraform Integration**: Uses Terraform, as IBM Cloud updates these automation modules regularly
- **Resource Naming**: Consistent `rhaii-{guid}-{type}` naming pattern
- **Comprehensive Logging**: Detailed terraform logs for troubleshooting
- **Clean State Management**: Automatic state refresh and cleanup

## Authentication Model

This role uses a pre-provisioned **IBM Cloud API key + existing resource group** model:

- `ibmcloud_api_key` authenticates directly to IBM Cloud. It must already have access to the resource group named by
  `ibmcloud_resource_group_name` (e.g. `Viewer`/`Editor` at the resource group level)
- An IBM Cloud Trusted Profile is configured, binding the SAML user to the instance by matching their email address.
  Note that this means that IBM Cloud sign-ins need to happen with the _exact same email address_ recorded by RHDP. This
  is typically your Kerberos account, not an alias.

## Requirements

- **IBM Cloud Account**: Valid IBM Cloud account with an existing resource group
- **IBM Cloud API Key**: An API key that already has:
  - access to the target resource group (e.g. Viewer/Editor role on the resource group), and
  - access to create Cloud Object Storage and RHAII service instances in that resource group
- **Terraform**: Automatically installed if not present (v1.9.8)
- **Ansible Collections**:
  - `community.general` (for the `terraform` module)

## Role Variables

### Required Variables

| Variable                       | Description                                                                                                           | Example                   |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| `ibmcloud_api_key`             | IBM Cloud API key for authentication. Must already have access to the resource group and Cloud Object Storage service | `your-api-key-here`       |
| `ibmcloud_resource_group_name` | Name of an **existing** IBM Cloud resource group to deploy resources into                                             | `my-project-rg`           |
| `guid`                         | Unique identifier for deployment                                                                                      | `user01`                  |
| `output_dir`                   | Directory for terraform files and logs                                                                                | `/tmp/instructlab-deploy` |

**Note**: Region is automatically set to `us-east` as RHAII service is only available in that region.

### Optional Variables (defaults/main.yml)

| Variable                                | Default                    | Description                          |
| --------------------------------------- | -------------------------- | ------------------------------------ |
| `ibmcloud_terraform_version`            | `1.9.8`                    | Terraform version to install         |
| `ibmcloud_provider_version`             | `1.80.4`                   | IBM Cloud Terraform provider version |
| `ibmcloud_terraform_name_prefix`        | `instructlab`              | Prefix for resource names            |
| `ibmcloud_storage_class`                | `standard`                 | Storage class for COS resources      |
| `ibmcloud_instructlab_instance_service` | `instructlab`              | RHAII service name                   |
| `ibmcloud_instructlab_instance_plan`    | `instructlab-pricing-plan` | Service plan for RHAII               |

## Resource Naming

All resources follow the pattern: `instructlab-{guid}-{type}`

Examples:

- COS Instance: `instructlab-user01-cos`
- COS Bucket: `instructlab-user01-bucket-<timestamp>`
- RHAII Service Instance: `instructlab-user01-il`

## Usage

### Basic Deployment

```yaml
- name: Deploy IBM RHAII Service
  hosts: localhost
  connection: local
  become: false
  vars:
    ACTION: provision
    ibmcloud_api_key: "{{ vault_ibmcloud_api_key }}"
    ibmcloud_resource_group_name: "my-project-rg"
    guid: "user01"
    output_dir: "/tmp/instructlab-{{ guid }}"
  roles:
    - rhpds.ibm_workloads.ibm_instructlab_service
```

### Complete Lifecycle Example

```yaml
---
- name: IBM RHAII Service Management
  hosts: localhost
  connection: local
  become: false
  vars:
    ibmcloud_api_key: "{{ vault_ibmcloud_api_key }}"
    ibmcloud_resource_group_name: "my-project-rg"
    guid: "{{ student_name | default('demo01') }}"
    output_dir: "/tmp/instructlab-{{ guid }}"

    # Optional customizations
    ibmcloud_terraform_name_prefix: "instructlab"

  tasks:
    # Provision RHAII Service
    - name: Deploy RHAII Service
      include_role:
        name: rhpds.ibm_workloads.ibm_instructlab_service
      vars:
        ACTION: provision

    # Later: Destroy RHAII Service
    - name: Clean up RHAII Service
      include_role:
        name: rhpds.ibm_workloads.ibm_instructlab_service
      vars:
        ACTION: destroy
      when: cleanup_resources | default(false)
```

## Actions

The role supports two primary actions controlled by the `ACTION` variable:

### `ACTION: provision`

- Looks up the existing resource group
- Deploys the COS instance/bucket and RHAII service instance
- Creates the RHAII -> COS authorization policy
- Surfaces the project link for integration

### `ACTION: destroy`

- Safely destroys all created resources
- Cleans up terraform state files
- Handles missing state gracefully
- Provides cleanup verification

## Logging and Troubleshooting

### Log Files

All operations generate detailed logs in the `output_dir`:

- `terraform_deploy_<timestamp>.log` - Deployment terraform logs
- `terraform_destroy_<timestamp>.log` - Destruction terraform logs
- `terraform.tfstate` - Terraform state file

### Common Issues

1. **Resource Naming Conflicts**
   - Use a different `guid` value
   - Run with `ACTION: destroy` first to clean up

2. **API Key / Resource Group Issues**
   - Verify `ibmcloud_resource_group_name` exists and is spelled correctly
   - Verify `ibmcloud_api_key` has access to that resource group and to the Cloud Object Storage service
   - Check key hasn't expired

3. **Region Availability**
   - Verify RHAII service is available in selected region
   - Check IBM Cloud service status

4. **Quota Limits**
   - Review IBM Cloud account quotas
   - Contact IBM support for quota increases

## Dependencies

This role has no external role dependencies but requires:

- An existing IBM Cloud resource group with appropriate service quotas
- Network connectivity to IBM Cloud APIs
- Sufficient disk space for terraform state and logs

## Error Handling

The role includes comprehensive error handling:

- **Retry Logic**: Automatic retries for transient failures
- **State Validation**: Terraform state refresh before operations
- **Graceful Degradation**: Continues with warnings when appropriate
- **Detailed Logging**: All operations logged for troubleshooting

## Security Considerations

- **API Key Protection**: Store API keys in Ansible Vault
- **Resource Isolation**: Each deployment uses unique GUID-based naming
- **Least Privilege**: Grant the API key only the access it needs on the target resource group and the Cloud Object
  Storage service
- **State Security**: Terraform state contains sensitive information

## License

Apache-2.0

## Author Information

- **Patrick Rutledge** - Red Hat
- **Tony Kay** - Red Hat

Ported from AgnosticD's `agnosticd.ibm.ibm_instructlab_service` role to the `rhpds.ibm_workloads` collection, replacing
the resource-group-creation / trusted-profile / SAML authentication model with a direct API key + existing resource
group model.
