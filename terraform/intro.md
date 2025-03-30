# Introduction to Terraform 

*   **Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently.**
*   It falls under the category of **Infrastructure as Code (IaC) tools .**
*   IaC allows you to define your entire cloud infrastructure as a set of config files .
*   Terraform interacts with cloud provider APIs to provision and manage resources on your behalf .
*   The course assumes familiarity with basic programming and Amazon Web Services (AWS) .
*   The course covers a progression from basics to modular, automated infrastructure code deploying to staging and production .
*   It alternates between theory and hands-on examples .
*   A reference architecture of a basic web application on AWS is used throughout the course, including EC2, ELB, RDS, S3, and Route 53 within the default VPC .
*   The choice of AWS is for simplicity, **Terraform is cloud-agnostic and can interact with anything with an API .**
*   A companion GitHub repository contains the code examples organised by module .


# Infrastructure as Code (IaC)

*   IaC allows you to define your entire infrastructure within your codebase .
*   Benefits of IaC :
    *   Knowing exactly what is provisioned at any time .
    *   Easy to reason about the state of your infrastructure .
    *   Confidence that multiple environments (staging, production) are the same .
    *   Using the power of programming languages for configuration .
*   Categories of IaC tools :
    *   Ad hoc scripts (e.g., shell scripts) - borderline IaC .
    *   Configuration Management tools (Ansible, Puppet, Chef) - focus on software configuration, more suited for on-prem .
    *   Server Templating tools (e.g., Packer for AMIs) - building templates for server images .
    *   Orchestration tools (e.g., Kubernetes) - defining application deployment, less about the underlying servers .
    *   **Provisioning tools (e.g., Terraform) - focused on provisioning cloud resources .**
*   **Declarative vs. Imperative tools :**
    *   **Declarative (e.g., Terraform): Define the desired end state, and the tool manages how to achieve it .**
    *   Imperative: Tell the system what to do and the sequence .
*   **Cloud-specific vs. Cloud-agnostic tools:**
    *   Cloud-specific (CloudFormation, Azure Resource Manager, Google Cloud Deployment Manager) - work within a single cloud .
    *   **Cloud-agnostic (Terraform, Pulumi) - can be used across multiple clouds and services with APIs .** Useful for multi-cloud or third-party services .

# Terraform Overview and Setup

*   **Terraform is a tool for building, changing, and versioning infrastructure safely and easily .** (Reiterated from HashiCorp)
*   **Why Terraform?** Enables applying software development best practices (version control, code reviews) to infrastructure .
*   **Cloud-agnostic and compatible with many clouds and services with APIs .**
*   Terraform can be used with other IaC tools :
    *   Terraform (provisioning) + Ansible (configuration management) .
    *   Terraform (provisioning) + Packer (image templating) .
    *   Terraform (provisioning) + Kubernetes (orchestration).
*   **Terraform Architecture :**
    *   **Terraform Core:** The engine that parses config and state .
    *   **Terraform State File:** Contains the current state of provisioned infrastructure .
    *   **Providers:** Plugins that tell Terraform how to interact with specific cloud provider APIs . Many providers are available .
*   **Installation :** Different methods for various operating systems (Homebrew, Chocolatey, direct binary download) .
*   **AWS Authentication for Terraform:**
    *   Create an IAM user with necessary permissions .
    *   Configure AWS CLI with access key ID, secret access key, and region (`aws configure`). Terraform uses these credentials by default .
*   **Basic Terraform Configuration (main.tf) :**
    *   `terraform` block to specify required providers (and versions).
    *   `provider` block to configure the provider (e.g., AWS region).
    *   `resource` block to define infrastructure resources (e.g., `aws_instance`) with attributes (AMI, instance type) .

# Basic Terraform Usage

*   **General workflow:**
    1.  **`terraform init`:** Initialises the project, downloads providers and modules.
    2.  **`terraform plan`:** Shows the changes that will be applied to reach the desired state.
    3.  **`terraform apply`:** Provisions or modifies the infrastructure.
    4.  **`terraform destroy`:** Tears down all the resources managed by Terraform.
*   **Terraform Provider Registry (registry.terraform.io) :** A list of available providers (official and community) . Official providers have a higher level of trust .
*   Providers are used in code by specifying them in the `required_providers` block within the `terraform` block and can be version-pinned . Each provider may have required configuration (e.g., region for AWS) .

# Terraform Init Deep Dive

*   When you run `terraform init` in a directory with Terraform code:
    *   It downloads the providers specified in the `terraform` block from the Terraform Registry or other configured registries.
    *   The provider code is stored in a `.terraform` hidden directory within the working directory.
    *   A **lock file** is created (`.terraform.lock.hcl`) which contains information about the specific provider versions and dependencies used.
    *   If any modules are used in the configuration, `terraform init` will also download those into a `.terraform/modules` subdirectory.

# Terraform State

*   **The Terraform state file is a JSON file that Terraform uses to understand the current state of your infrastructure.**
*   It contains metadata about every resource (e.g., IP address, ARN for an AWS instance) and data object provisioned by Terraform.
*   **State files can contain sensitive information (e.g., database passwords).** Therefore, they need to be protected with encryption and proper permissions.
*   State can be stored **locally (default)** or **remotely.**
*   **Local Backend:**
    *   Pros: Super easy to get started, works out of the box.
    *   Cons: Sensitive values in plain text, uncollaborative (hard to share state), manual operation.
*   **Remote Backend:**
    *   Separates the developer from the state file stored on a remote server.
    *   Options:
        *   **Terraform Cloud (managed by HashiCorp):** Hosts state files, manages permissions. Free for up to five users, then costs per user.
        *   **Self-managed (e.g., AWS S3 and DynamoDB):** S3 bucket to store the state file (encryption recommended), DynamoDB table for state locking (prevents concurrent applies).
    *   Benefits of Remote Backend:
        *   Encrypts sensitive data and gets it off local systems.
        *   Enables collaboration with multiple engineers.
        *   Allows for automation through CI/CD pipelines.
    *   **Bootstrapping Remote Backend:**
        1.  Configure Terraform with no remote backend (local by default).
        2.  Define the S3 bucket and DynamoDB table resources in your Terraform code. The DynamoDB table's hash key must be `LockID`.
        3.  Run `terraform apply` to provision these backend resources.
        4.  Update your Terraform configuration to use the remote backend (S3 type, specify bucket and DynamoDB table).
        5.  Run `terraform init` again. Terraform will detect the backend change and prompt to import the local state to the remote backend.

# Terraform Plan and Apply Deep Dive

*   **`terraform plan`:**
    *   Takes your Terraform configuration (desired state).
    *   Compares it with the Terraform state (actual deployed state).
    *   Determines the necessary actions (create, update, delete) to make the actual state match the desired state.
    *   The plan can be reviewed before applying.
*   **`terraform apply`:**
    *   Takes the execution plan generated by `terraform plan`.
    *   Makes the necessary API calls to the cloud provider to provision or modify the resources.
    *   Updates the Terraform state file to reflect the new state of the infrastructure.

# Terraform Destroy Deep Dive

*   **`terraform destroy`:**
    *   Reverses the actions of `terraform apply`.
    *   Deletes all the resources managed by the current Terraform configuration.
    *   **Use this command with extreme caution, especially in live production environments.** It's typically used for cleaning up test environments or decommissioning infrastructure.

# Remote Backends in Detail

*   **Terraform Cloud Backend:**
    *   Managed service by HashiCorp.
    *   Configured in the `terraform` block with `backend "remote"` and specifying `organization` and `workspace`.
    *   Web UI for managing state, permissions, etc..
    *   Free for up to 5 users, then $20/user/month.
*   **Self-managed AWS Backend:**
    *   Uses an **S3 bucket** to store the state file. Encryption should be enabled.
    *   Uses a **DynamoDB table** for state locking to prevent concurrent modifications. The hash key must be `LockID`.
    *   Requires a bootstrapping process to manage the backend resources with Terraform.

# Building the Reference Architecture

*   Demonstrates building the web application architecture using Terraform.
*   Uses a remote backend (pre-provisioned S3 bucket and DynamoDB table).
*   **Key resources provisioned:**
    *   Two EC2 instances (same Ubuntu AMI, t2.micro, security groups). Simple web server (Python) in user data.
    *   S3 bucket (named, server-side encryption enabled).
    *   Data blocks to reference the default VPC and subnet.
    *   Security groups (allowing inbound traffic on port 8080 for instances).
    *   Elastic Load Balancer (listener on port 80, target group for EC2 instances, listener rule). Separate security group for the load balancer (inbound on port 80).
    *   Route 53 DNS zone for a domain (e.g., devopsdeploy.com) and an A record pointing to the load balancer.
    *   RDS database instance (provisioning shown, but not application connection).
*   `terraform init`, `terraform plan`, and `terraform apply` commands are run to provision the infrastructure.
*   Verification of the deployed web application via the load balancer's DNS name and the custom domain.
*   Emphasis on cleaning up resources with `terraform destroy`.
*   Notes that the initial configuration in `main.tf` is large and would be broken down into smaller files and use variables in a real-world scenario.

# Variables and Outputs

*   **Variables (Input Variables):**
    *   Used to make Terraform configurations more flexible and reusable.
    *   Referenced using `var.variable_name`.
    *   Declared in `variable` blocks with a name and optionally a type and default value.
    *   Like input parameters to a function.
*   **Local Variables:**
    *   Defined within `locals` block (plural) and referenced using `local.variable_name`.
    *   Like temporary variables within a function's scope.
    *   Useful for reusing values within a configuration.
*   **Output Variables:**
    *   Declared in `output` blocks with a name and a value (expression).
    *   Like the return value of a function.
    *   Allow you to expose values from your infrastructure (e.g., IP addresses).
*   **Setting Input Variable Values:** Order of precedence (lowest to highest):
    1.  Prompted at the command line (if no default).
    2.  Default values in the `variable` block.
    3.  Environment variables (`TF_VAR_<variable_name>`).
    4.  `terraform.tfvars` file.
    5.  `*.auto.tfvars` files.
    6.  `-var` and `-var-file` options with `terraform plan` or `terraform apply`.
*   **Variable Types:** Primitive (string, number, boolean) and complex (list, set, map, object, tuple).
*   **Validation:** Automatic type checking. Custom validation rules can be defined in the `variable` block using the `validation` argument.
*   **Sensitive Data:** Mark sensitive variables with `sensitive = true` in the `variable` block. Avoid putting sensitive values in `.tfvars` files; use environment variables or `-var` flags, potentially retrieving from secret stores. Sensitive outputs are masked in the Terraform plan.
*   **Examples:** Demonstrates defining variables in `variables.tf`, providing default values in `terraform.tfvars`, and passing sensitive values via `-var`. Outputs for instance and database IPs are shown.
*   **Refactoring Web App with Variables:** Shows how the hardcoded values in the web app configuration are replaced with variables (region, instance type, bucket name, domain, database credentials). This allows for deploying multiple similar environments by just changing variable values.

# Advanced HCL Features

*   **Expressions:**
    *   Template strings (string interpolation).
    *   Operators (arithmetic, comparison, logical).
    *   Conditionals (ternary syntax).
    *   Collections (for loops for lists, splat operator for expanding lists).
    *   Dynamic blocks.
    *   Type and version constraints.
*   **Functions:** Built-in functions for math, date/time, hash and crypto, string manipulation, collections, etc.. Refer to Terraform documentation for details.
*   **Meta-Arguments:** Arguments available within resource blocks that control resource behaviour.
    *   **`depends_on`:** Explicitly define dependencies between resources when Terraform can't infer them (e.g., instance depends on a role policy).
    *   **`count`:** Create multiple identical copies of a resource. `count.index` can be used to differentiate instances (e.g., in tags).
    *   **`for_each`:** Create multiple similar resources based on an iterable (map or set), providing more control over individual instances.
    *   **`lifecycle`:** Control the lifecycle of resources:
        *   `create_before_destroy`: Create a new resource before destroying the old one (for zero-downtime updates).
        *   `ignore_changes`: Tell Terraform to ignore changes to specific attributes after creation (useful for provider-managed metadata).
        *   `prevent_destroy`: Prevent accidental deletion of a resource.

# Provisioners

*   **Provisioners allow you to execute actions on a local or remote machine after a resource is created.**
*   Different types of provisioners:
    *   `file`: Copy files onto a remote resource.
    *   `local-exec`: Execute a command on the machine running Terraform.
    *   `remote-exec`: Execute a command on the newly created resource (requires SSH or other remote access).
    *   Vendor-specific provisioners (e.g., for Ansible) may also exist.
*   Use cases:
    *   Running a startup script on an EC2 instance.
    *   Integrating with configuration management tools like Ansible.
*   Provisioners should be used sparingly as they can make infrastructure management less predictable. Consider using cloud-init or baked-in configurations instead where possible (note: this last part is implied but not explicitly stated in the source).

# Project Organisation and Modules

*   **A module is a container for multiple resources that are bundled together for reusability.**
*   Consists of a collection of `.tf` or `.tf.json` files within a single directory.
*   The root module is the main directory containing your Terraform configuration.
*   Child modules are referenced from the root module.
*   **Benefits of using modules:**
    *   Raises the level of abstraction.
    *   Allows infrastructure specialisation.
    *   Promotes reusability across projects and environments.
    *   Hides complexity from application developers.
    *   Encapsulates best practices.
*   **Module Sources:**
    *   Local paths (relative paths to directories on the local filesystem).
    *   Terraform Registry (public and private registries). Source format: `<namespace>/<name>/<provider>`. Version pinning is important.
    *   Git repositories (HTTPS or SSH URLs to GitHub or other Git hosts). Specific syntax for GitHub. Version pinning via tags/commits.
    *   S3 buckets (less common).
*   **Referencing Modules:** Use the `module` block with a `source` argument specifying the location.
*   **Module Inputs:** Input variables defined in the module can be passed in the `module` block. This allows for customising module behaviour.
*   **Meta-Arguments in Modules:** `count`, `for_each`, `providers`, and `depends_on` can also be used within `module` blocks.
*   **What Makes a Good Module?:**
    *   Raises the abstraction level from basic resources.
    *   Groups logically related resources.
    *   Exposes necessary input variables.
    *   Provides useful default values.
    *   Returns relevant outputs.
*   **Terraform Registry:** Contains both providers and modules. Many modules available for various providers and use cases (e.g., a security group module).


# Managing Multiple Environments

*   The need to manage different environments (dev, staging, production) with similar configurations.
*   **Two main approaches:**
    *   **Terraform Workspaces:**
        *   Use named sections within a single remote backend to manage different environments.
        *   Managed with the `terraform workspace` command.
        *   Can reference the current workspace using `terraform.workspace` in configurations (e.g., for naming resources).
        *   Pros: Easy to get started, minimises code duplication.
        *   Cons: Easy to forget the current workspace and apply changes to the wrong environment, state files are all in the same remote backend (potential permission issues), less explicit about what is deployed in each environment just by looking at the code.
    *   **Directory Structure:**
        *   Create separate subdirectories for each environment (e.g., `dev`, `staging`, `production`).
        *   Each directory contains its own Terraform configuration (potentially consuming shared modules).
        *   Pros: Isolated backends (different S3 buckets, DynamoDB tables) for each environment, reduces potential for human error (less chance of applying to the wrong environment), fully represents the deployed state in the codebase.
        *   Cons: More code duplication compared to workspaces, requires navigating between directories to apply changes.
*   As infrastructure grows, consider breaking down configurations into logical component groups (e.g., compute, networking) rather than one massive config.
*   **Terraform Remote State:** Allows referencing the state (outputs) of a completely separate Terraform configuration. Useful for decoupling infrastructure components while still being able to share information (e.g., referencing the IP addresses of compute instances from a networking configuration).
*   **Meta-tooling (TerraGrant):** Tools like TerraGrant can help manage complexity with the directory structure approach, reduce code repetition, and simplify operations across multiple environments and accounts.

# Testing Infrastructure as Code

*   **Code Rot:** The concept that infrastructure code can degrade over time due to out-of-band changes, unpinned versions, deprecated resources, and unapplied changes.
*   **Testing prevents code rot.**
*   **Types of Tests:**
    *   **Static Checks:**
        *   `terraform fmt`: Opinionated code formatting. `terraform fmt check` to verify.
        *   `terraform validate`: Checks for syntactical correctness and required arguments.
        *   `terraform plan`: Shows potential changes, can be used to detect out-of-band modifications.
    *   **Custom Validation Rules:** Define constraints on variable values within the `variable` block using the `validation` argument.
    *   **Third-Party Tools:** TFLint (additional linting), Checkov/TFSec/Terascan (security scanning), Terraform Sentinel (enterprise policy enforcement).
    *   **Manual Testing:** Running `init`, `plan`, `apply`, `destroy` manually.
    *   **Automated Testing:**
        *   **Bash Scripts:** Automating the manual testing cycle (e.g., apply, wait, curl endpoint, destroy). A basic example is shown.
        *   **Terratest (Go):** Framework for writing infrastructure tests in Go, allowing for more complex assertions and retries. An example testing HTTP endpoint availability is demonstrated.
*   **Repo Layout for Testing:** Suggested structure: `modules/`, `examples/`, `deployed/`, `test/`.
*   **Automated Testing with GitHub Actions:** Running tests as part of a CI/CD pipeline. Periodically running `terraform plan` in CI to detect drift.

# Developer Workflows and Automation

*   **General Developer Workflow:** Write/update code -> local testing -> create Pull Request (PR) -> code review -> CI tests (e.g., Terratest) -> merge PR -> CD pipeline to staging (automated) -> promotion to production (potentially manual or automated after tagging release).
*   **Testing Schedule:** Run `terraform plan` periodically in CI to detect out-of-band changes.
*   **Working with Multiple Accounts:** Beneficial for security, isolation, and avoiding naming conflicts. Terminology varies by cloud (AWS accounts, GCP projects). Adds complexity but generally worth it. Reference to a HashiCorp talk on multi-account AWS with Terraform.
*   **Third-Party Tools:**
    *   TerraGrant: Minimise code repetition, helps with multi-account setups.
    *   Cloud Nuke: Easily clean up all resources in a cloud account (useful for test environments).
*   **Local Scripting:** Use Makefiles or shell scripts to store and run frequently used Terraform commands to reduce human error.
*   **CI/CD Tools:** GitHub Actions (demonstrated later), CircleCI, GitLab CI, Atlantis (Terraform-specific) are all viable options for automating IaC workflows.

# Terraform Gotchas

*   **Changing Resource Names:** Can cause Terraform to delete the old resource and create a new one. Be careful when renaming.
*   **Sensitive Data in State Files:** Requires encryption and careful permission management.
*   **Cloud Timeouts:** Terraform might time out if cloud resource provisioning takes too long. Timeouts can be configured, and re-running `terraform apply` often resolves this.
*   **Naming Conflicts:** Resources within the same cloud account often require unique names, leading to potential conflicts if not managed properly.
*   **Forgetting to Destroy Test Infrastructures:** Can lead to unexpected cloud bills. Tools like Cloud Nuke can help.
*   **Unidirectional Version Upgrades:** State files are associated with the Terraform version used. Upgrading Terraform might prevent using older versions with the same state. Ensure team consistency in Terraform version.
*   **Multiple Ways to Accomplish the Same Configuration:** Choose the cleanest and most maintainable approach.
*   **Immutable Parameters:** Some resource parameters cannot be changed after creation, requiring deletion and recreation to modify.
*   **Out-of-Band Changes:** Making changes to infrastructure outside of Terraform will lead to the Terraform state becoming inaccurate and potentially dangerous if `terraform apply` is run.
