**Current State:**
- PostgreSQL 14 (`db_engine_version = "14"`, `rds_family = "postgres14"`)
- Module version: 8.1.0
- `allow_major_version_upgrade = "true"` (already enabled)
- Development environment (`is_production = "false"`)

**Target State:**
- PostgreSQL 16 (latest stable version)
- Following best practices for production readiness

# Database Upgrade Plan for hmpps-community-accommodation-dev

## 📋 Pre-Upgrade Assessment

### 1. **Current Configuration Analysis**
- **Database Engine**: PostgreSQL 14
- **RDS Family**: postgres14
- **Module Version**: 8.1.0
- **Environment**: Development (non-production)
- **Major Version Upgrades**: Already enabled (`allow_major_version_upgrade = "true"`)

### 2. **Compatibility Check**
- ✅ PostgreSQL 14 → 16 is a supported upgrade path
- ✅ Module version 8.1.0 supports PostgreSQL 16
- ✅ Development environment allows for testing

## 🚀 Step-by-Step Upgrade Procedure

### **Phase 1: Pre-Upgrade Preparation**

#### Step 1.1: Create Backup Snapshot
```bash
# Connect to the cluster and create a manual snapshot
kubectl -n hmpps-community-accommodation-dev run shell --rm -i --tty --image bitnami/postgresql -- bash

# Get database credentials
kubectl -n hmpps-community-accommodation-dev get secret rds-postgresql-instance-output -o yaml | ./decode.rb

# Create manual snapshot (document the snapshot ID)
# Note: This will be done via AWS Console or CLI
```

#### Step 1.2: Document Current State
```bash
# Get current database information
kubectl -n hmpps-community-accommodation-dev get configmap rds-postgresql-instance-output -o yaml

# Document current settings for rollback reference
```

#### Step 1.3: Notify Team
- Inform the development team about the planned upgrade
- Schedule the upgrade during low-usage hours
- Ensure team is available for testing post-upgrade

### **Phase 2: Infrastructure Changes**

#### Step 2.1: Update Terraform Configuration



#### Step 2.2: Plan Terraform Changes
```bash
# Navigate to the resources directory
cd namespaces/live.cloud-platform.service.justice.gov.uk/hmpps-community-accommodation-dev/resources

# Initialize Terraform
terraform init

# Plan the changes to review what will be modified
terraform plan
```

**Expected Changes:**
- `db_engine_version`: "14" → "16"
- `rds_family`: "postgres14" → "postgres16"
- `prepare_for_major_upgrade`: false → true
- `deletion_protection`: null → false
- `skip_final_snapshot`: null → "false"

### **Phase 3: Execute Upgrade**

#### Step 3.1: Apply Infrastructure Changes
```bash
# Apply the Terraform changes
terraform apply

# Monitor the upgrade process
# This will trigger a major version upgrade which may take 30-60 minutes
```

#### Step 3.2: Monitor Upgrade Progress
```bash
# Check RDS instance status
kubectl -n hmpps-community-accommodation-dev get configmap rds-postgresql-instance-output -o yaml

# Monitor AWS RDS console for upgrade progress
# Look for: "Upgrading" → "Available" status
```

### **Phase 4: Post-Upgrade Validation**

#### Step 4.1: Verify Database Version
```bash
# Connect to the upgraded database
kubectl -n hmpps-community-accommodation-dev run shell --rm -i --tty --image bitnami/postgresql -- bash

# Get new credentials
kubectl -n hmpps-community-accommodation-dev get secret rds-postgresql-instance-output -o yaml | ./decode.rb

# Connect and verify version
psql -h [rds_instance_address] -U [database_username] [database_name]
# Run: SELECT version();
# Expected: PostgreSQL 16.x
```

#### Step 4.2: Application Testing
```bash
# Test application connectivity
# Verify all application features work correctly
# Check for any compatibility issues
```

#### Step 4.3: Performance Validation
```bash
# Monitor database performance
# Check for any performance regressions
# Verify all queries execute correctly
```

### **Phase 5: Post-Upgrade Cleanup**

#### Step 5.1: Re-enable Safety Features



#### Step 5.2: Apply Final Configuration
```bash
# Apply the cleanup changes
terraform apply
```

## �� Rollback Plan

### **If Upgrade Fails:**

#### Option 1: Restore from Snapshot
```bash
# Create new RDS instance from backup snapshot
# Update Terraform to use snapshot_identifier
# Apply changes to restore previous state
```

#### Option 2: Revert Terraform Changes
```bash
# Revert the Terraform configuration to PostgreSQL 14
# Apply changes to downgrade
# Note: This may require manual intervention
```

## 📊 Monitoring and Validation Checklist

### **Pre-Upgrade:**
- [ ] Manual snapshot created
- [ ] Team notified
- [ ] Application state documented
- [ ] Terraform plan reviewed

### **During Upgrade:**
- [ ] Upgrade progress monitored
- [ ] No application errors
- [ ] Database connectivity maintained

### **Post-Upgrade:**
- [ ] Database version verified (PostgreSQL 16)
- [ ] Application functionality tested
- [ ] Performance validated
- [ ] Safety features re-enabled
- [ ] Documentation updated

## 🚨 Important Notes

### **Timing Considerations:**
- **Upgrade Duration**: 30-60 minutes (major version upgrade)
- **Downtime**: Minimal (RDS handles this automatically)
- **Best Time**: During low-usage hours

### **Risk Mitigation:**
- Development environment allows for safe testing
- Manual snapshot provides rollback option
- `allow_major_version_upgrade = "true"` already enabled
- Performance insights enabled for monitoring

### **Production Considerations:**
When applying this to production:
1. **Schedule during maintenance window**
2. **Enable deletion_protection = true**
3. **Create multiple backup snapshots**
4. **Test rollback procedures**
5. **Monitor application performance closely**
6. **Have team on standby for issues**

## 📝 Documentation Updates

After successful upgrade:
1. Update team documentation
2. Document lessons learned
3. Update production upgrade plan
4. Share findings with other teams

This plan follows the best practices outlined in the RDS documentation and incorporates safety measures appropriate for a development environment while establishing a foundation for production upgrades.
