# Blue-Green Deployment Beta Environment - User Guide

This document describes how to use the blue-green deployment system for the beta environment, which is now fully operational.

## Overview

The beta environment uses a blue-green deployment strategy with:
- Two ASGs: `beta-blue` and `beta-green`
- Two target groups: `Beta-Blue` and `Beta-Green`
- ALB listener rule switching for `/beta*` traffic
- State tracking via SSM parameters

## Prerequisites

- AWS credentials with appropriate permissions
- Access to the CE CLI (`ce` command)
- Beta environment must be available

## Basic Usage

### Check Current Status

```bash
ce --env beta blue-green status
```

Example output:
```
Blue-Green Status for beta:
Active Color: blue
Inactive Color: green

ASG Status:
  blue (ACTIVE):
    ASG Name: beta-blue
    Desired/Min/Max: 1/0/4
    ASG Health: 1/1 healthy
    Target Group: 1/1 healthy ✅
  green:
    ASG Name: beta-green
    Desired/Min/Max: 0/0/4
    ASG Health: 0/0 healthy
    Target Group: 0/0 healthy ❓
```

### Deploy New Version

```bash
# Deploy to inactive color (automatically determined)
ce --env beta blue-green deploy

# Deploy with specific capacity
ce --env beta blue-green deploy --capacity 2

# Skip confirmation prompts (for automation)
ce --env beta blue-green deploy --skip-confirmation
```

The deployment process:
1. Identifies active (blue) and inactive (green) colors
2. Protects active ASG from scaling during deployment
3. Scales up inactive ASG with new instances
4. Waits for health checks to pass
5. Switches ALB listener rule to point to new target group
6. Cleans up and restores ASG settings

### Manual Operations

```bash
# Switch to specific color (if instances already exist)
ce --env beta blue-green switch green

# Rollback to previous color
ce --env beta blue-green rollback

# Clean up inactive ASG (scale to 0)
ce --env beta blue-green cleanup

# Shut down entire environment
ce --env beta blue-green shutdown

# Validate infrastructure setup
ce --env beta blue-green validate
```

## Advanced Usage

### Deployment with Existing Instances

If the inactive ASG already has instances, the system will warn you:

```
⚠️  WARNING: The green ASG already has 1 instance(s) running!
This means you'll be switching to existing instances rather than deploying fresh ones.

If you want to:
  • Switch traffic to existing green instances → use 'ce --env beta blue-green switch'
  • Roll back to green → use 'ce --env beta blue-green rollback'
  • Deploy fresh instances → run 'ce --env beta blue-green cleanup' first, then deploy

Do you want to continue with deployment to existing green instances? (yes/no):
```

### Automation-Friendly Commands

```bash
# Deploy without any confirmation prompts
ce --env beta blue-green deploy --skip-confirmation

# Switch without confirmation
ce --env beta blue-green switch green --skip-confirmation

# Shutdown without confirmation
ce --env beta blue-green shutdown --skip-confirmation
```

## Safety Features

### ASG Protection During Deployment

During deployment, the active ASG is protected from unwanted scaling:
- MinSize and MaxSize are temporarily set to current capacity
- Prevents CloudWatch alarms from scaling up
- Prevents autoscaling policies from scaling down
- Protection is automatically restored after deployment

### Signal Handling

If a deployment is interrupted (Ctrl+C, SIGTERM), the system:
- Detects the interruption
- Restores original ASG capacity settings
- Resets inactive ASG to min size 0
- Exits cleanly without orphaned resources

### Confirmation Prompts

Most operations require confirmation unless `--skip-confirmation` is used:
- Deployment operations
- Traffic switching
- Environment shutdown
- ASG cleanup

## Monitoring and Troubleshooting

### Health Check Types

The status command shows multiple health indicators:

1. **ASG Health**: EC2 instance health from ASG perspective
2. **Target Group Health**: ALB target group health status
   - ✅ Healthy and receiving traffic
   - 🟡 Healthy but not receiving traffic (standby)
   - ❌ Unhealthy or failing health checks
   - ❓ Unknown state
3. **HTTP Health**: Direct HTTP endpoint checks (when run from admin node)

### Common Issues

#### Deployment Fails to Start
```bash
# Check if ASGs exist
ce --env beta blue-green validate

# Check current status
ce --env beta blue-green status
```

#### Health Checks Failing
```bash
# Check detailed instance information
ce --env beta blue-green status --detailed

# Check ASG scaling activities
aws autoscaling describe-scaling-activities --auto-scaling-group-name beta-blue
```

#### Traffic Not Switching
```bash
# Verify ALB listener rule configuration
ce --env beta blue-green validate

# Check SSM parameters
aws ssm get-parameter --name /compiler-explorer/beta/active-color
aws ssm get-parameter --name /compiler-explorer/beta/active-target-group-arn
```

### Manual Recovery

If something goes wrong, you can manually reset:

```bash
# Scale down both ASGs
aws autoscaling set-desired-capacity --auto-scaling-group-name beta-blue --desired-capacity 0
aws autoscaling set-desired-capacity --auto-scaling-group-name beta-green --desired-capacity 0

# Reset SSM parameters to known state
aws ssm put-parameter --name /compiler-explorer/beta/active-color --value blue --overwrite

# Start fresh
ce --env beta blue-green deploy --capacity 1
```

## Environment Lifecycle

### Typical Workflow

1. **Normal Operation**: One color active, other inactive
2. **Deploy New Version**: Scale up inactive, switch traffic
3. **Verify Deployment**: Check status and health
4. **Cleanup**: Scale down old version (optional)
5. **Next Deployment**: Process repeats with colors swapped

### Environment Shutdown

For cost savings or maintenance:

```bash
# Shutdown entire environment
ce --env beta blue-green shutdown

# Restart when needed
ce --env beta blue-green deploy --capacity 1
```

### Long-term Maintenance

```bash
# Periodic cleanup of inactive instances
ce --env beta blue-green cleanup

# Validate infrastructure health
ce --env beta blue-green validate
```

## Integration with Existing Workflows

### Build and Deploy

```bash
# Set the version to deploy
ce --env beta builds set_current <version>

# Deploy using blue-green strategy
ce --env beta blue-green deploy
```

### Monitoring Integration

The blue-green system integrates with existing monitoring:
- CloudWatch metrics for both ASGs
- ALB target group health monitoring
- SSM parameter change events

## Best Practices

1. **Always check status** before performing operations
2. **Use cleanup regularly** to avoid resource waste
3. **Validate infrastructure** after infrastructure changes
4. **Monitor health checks** during deployments
5. **Keep deployments small** for easier rollback
6. **Use skip-confirmation** in automation only
7. **Test rollback procedures** periodically

## Limitations

- Currently only available for beta environment
- Requires manual capacity planning
- Health checks must pass before traffic switch
- No gradual traffic shifting (atomic switch only)

## Future Enhancements

See GitHub issue #1653 for planned improvements including:
- Intentional shutdown state tracking
- Enhanced monitoring capabilities
- Production environment support
