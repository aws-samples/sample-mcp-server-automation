# MCP Server Automation CLI

A powerful CLI tool that automates the process of transforming Model Context Protocol (MCP) stdio servers into Docker images deployed on AWS ECS using [mcp-proxy](https://github.com/punkpeye/mcp-proxy). This tool bridges the gap between local MCP servers and remote HTTP-based deployments.

## 🚀 Features

- **🔄 Automatic Build**: Fetch MCP servers from GitHub, build Docker images, and push to ECR
- **☁️ One-Click Deploy**: Generate CloudFormation templates and deploy complete ECS infrastructure
- **🔍 Smart Detection**: Automatically detect MCP server commands from README files
- **🐳 Multi-Architecture**: Support for Python, Node.js, and hybrid MCP servers
- **🔧 Debug Support**: Built-in debug logging for troubleshooting
- **📝 Config Generation**: Generate MCP client configurations for Claude Desktop, Cline, etc.

## 📋 Prerequisites

- **Python 3.8+**
- **Docker** (with daemon running)
- **AWS CLI** configured with appropriate permissions
- **AWS ECR repository** (created if using ECR push)
- **AWS ECS cluster** (created if deploying)

## 📖 Quick Start

### Using uvx (Recommended)

The easiest way to use this tool is with `uvx`, which handles dependencies automatically:

```bash
# Install from a Git repository
uvx --from git+https://github.com/yubingjiaocn/mcp-server-automation mcp-server-automation --config your-config.yaml
```

### Local Development Setup

```bash
# Clone and setup
git clone <repository-url>
cd mcp-convert-automate

# Create virtual environment and install dependencies
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt

# Install in development mode (optional - for local CLI usage)
pip install -e .
```

## ⚙️ Configuration

The tool uses a unified YAML configuration file with `build` and `deploy` sections.

### Unified Configuration

```yaml
build:
  # Required: GitHub repository URL
  github_url: "https://github.com/awslabs/mcp"
  
  # Optional: Subfolder if MCP server is not in root
  subfolder: "src/aws-documentation-mcp-server"
  
  # Optional: Git branch to build from (default: main)
  branch: "develop"
  
  # Required for deployment: Must be true to enable ECR push and deployment
  push_to_ecr: true
  
  # Optional: Custom image URI (overrides auto-generation)
  # image_uri: "123456789012.dkr.ecr.us-west-2.amazonaws.com/custom-repo/my-server:v1.0"
  
  # Optional: Custom image name (auto-generated if not provided)
  # image_name: "my-mcp-server"
  
  # Optional: ECR repository (auto-generated if push_to_ecr=true)
  # ecr_repository: "123456789012.dkr.ecr.us-east-1.amazonaws.com/mcp-servers"
  
  # Optional: AWS region (default: from AWS profile, fallback to us-east-1)
  # aws_region: "us-west-2"
  
  # Optional: Custom Dockerfile path
  # dockerfile_path: "./custom.Dockerfile"
  
  # Optional: Override auto-detected MCP server command
  # command_override:
  #   - "python"
  #   - "-m"
  #   - "my_server_module"
  #   - "--verbose"
  
  # Optional: Set environment variables in the container
  # environment_variables:
  #   LOG_LEVEL: "debug"
  #   AWS_REGION: "us-east-1"
  #   MCP_SERVER_NAME: "custom-server"

deploy:
  # Required: Enable deployment (only works when push_to_ecr=true)
  enabled: true
  
  # Required: ECS service name
  service_name: "my-mcp-service"
  
  # Required: ECS cluster name
  cluster_name: "my-ecs-cluster"
  
  # Required: VPC ID where resources will be created
  vpc_id: "vpc-12345678"
  
  # Required: Subnet configuration 
  alb_subnet_ids:    # Public subnets for ALB (minimum 2 in different AZs)
    - "subnet-public-1"
    - "subnet-public-2"
  ecs_subnet_ids:    # Private subnets for ECS tasks (minimum 1, should resides in AZ of alb_subnet_ids)
    - "subnet-private-1"
    - "subnet-private-2"

  # Optional: Container port (default: 8000)
  port: 8000
  
  # Optional: Task CPU units (default: 256)
  cpu: 256
  
  # Optional: Task memory in MB (default: 512)
  memory: 512
  
  # Optional: SSL certificate ARN for HTTPS
  certificate_arn: "arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012"
  
  # Optional: Save MCP client configuration to file
  save_config: "./mcp-config.json"
```

## 🏗️ Architecture

### Build Process Flow

1. **Repository Analysis**: Downloads GitHub repos and detects MCP server configuration from README files
2. **Command Detection**: Parses JSON blocks in README files to extract MCP server start commands, prioritizing NPX/uvx over Docker commands
3. **Dockerfile Generation**: Uses Jinja2 templates to create multi-stage Docker builds with mcp-proxy CLI integration
4. **Image Building**: Creates hybrid Node.js + Python containers with proper dependency management

### Deployment Architecture

```
GitHub Repo → Docker Build → ECR → ECS Fargate ← ALB ← Internet
     ↓              ↓           ↓         ↓        ↓
MCP Server → mcp-proxy + MCP → Image → Service → HTTP/SSE Endpoints
```

### Key Technical Details

- **mcp-proxy Integration**: Uses TypeScript/Node.js CLI tool for HTTP transport with debug logging enabled
- **Container Architecture**: Multi-stage builds with `node:24-bullseye` base image, includes netcat for health checks
- **Command Format**: `mcp-proxy --debug --port 8000 --shell <command> [-- <args>]` for proper argument ordering
- **Transport Protocol**: Converts MCP stdio to HTTP with `/mcp` endpoint for Streamable HTTP transport
- **Dynamic Tagging**: Images tagged with git commit hash and timestamp (e.g., `a1b2c3d4-develop-20231222-143055`)
- **Branch Support**: Can build from specific git branches, defaults to 'main'
- **Health Checks**: Container uses netcat port checking, ALB health checks `/mcp` endpoint expecting HTTP 400
- **MCP Config Generation**: Automatically generates and prints MCP client configuration after deployment
- **Infrastructure**: Complete CloudFormation stack with VPC, ALB, ECS Fargate, security groups, and IAM roles

## 🔧 Advanced Usage

### Custom Dockerfile

```yaml
build:
  github_url: "https://github.com/my-org/custom-mcp-server"
  dockerfile_path: "./custom/Dockerfile"
  push_to_ecr: true
deploy:
  enabled: true
  # ... deployment configuration
```

### Command Override

Override the automatically detected MCP server command:

```yaml
build:
  github_url: "https://github.com/my-org/custom-mcp-server"
  command_override:
    - "python"
    - "-m"
    - "my_server_module"
    - "--verbose"
    - "--port"
    - "3000"
  push_to_ecr: true
```

### Environment Variables in Container

Set custom environment variables that will be available to the MCP server at runtime:

```yaml
build:
  github_url: "https://github.com/my-org/custom-mcp-server"
  environment_variables:
    LOG_LEVEL: "debug"
    AWS_REGION: "us-east-1"
    MCP_SERVER_NAME: "custom-server"
    PYTHONPATH: "/app/mcp-server:/custom/path"
  push_to_ecr: true
```

### System Environment Variables

Set environment variables to override default AWS settings:

```bash
export AWS_REGION=us-west-2
export ECS_CLUSTER_NAME=my-production-cluster
```

## 🐛 Troubleshooting

### Docker Build Issues
- Ensure Docker daemon is running
- Check that the MCP server has proper dependency files (requirements.txt, pyproject.toml, etc.)
- Verify GitHub repository URL is accessible

### ECR Push Issues
- Ensure AWS credentials have ECR permissions
- Verify ECR repository exists and is accessible
- Check that Docker is authenticated with ECR

### CloudFormation Deployment Issues
- Ensure AWS credentials have sufficient permissions
- Check that the ECS cluster exists
- Verify AWS region is correct
- Review CloudFormation events in AWS Console for detailed error messages

### MCP Server Connection Issues
- Check container logs: `docker logs <container-id>`
- Verify health check endpoint: `curl http://<alb-url>/mcp` (expects HTTP 400)
- Test direct connection: `curl http://<alb-url>/mcp`
- Use debug mode for detailed logging

## 🔐 AWS Permissions Required

The AWS credentials used must have the following permissions:

### ECR Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:GetAuthorizationToken",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    }
  ]
}
```

### ECS and CloudFormation Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:*",
        "cloudformation:*",
        "ec2:*",
        "elasticloadbalancing:*",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "logs:CreateLogGroup",
        "logs:DescribeLogGroups"
      ],
      "Resource": "*"
    }
  ]
}
```

## 📝 MCP Client Configuration

After deployment, the tool generates configuration for MCP clients:

```json
{
  "mcpServers": {
    "my-mcp-server": {
      "type": "sse",
      "url": "http://<ALB address>/sse"
    }
  }
}
```

### Testing MCP Connection

```bash
# Install mcp-proxy client
npm install -g mcp-proxy

# Test connection
mcp-proxy https://your-alb-url.amazonaws.com/mcp
```

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request

## 📄 License

This project is licensed under the MIT-0 License.

## 🆘 Support

- Check the [troubleshooting section](#-troubleshooting) for common issues
- Review CloudFormation events in AWS Console for deployment issues
- Use debug mode for detailed logging
- Open an issue for bugs or feature requests