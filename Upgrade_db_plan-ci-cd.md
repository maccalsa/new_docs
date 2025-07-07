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

### **Phase 2: Infrastructure Changes (Git Workflow)**

#### Step 2.1: Create Feature Branch
```bash
# Clone the repository (if not already done)
git clone https://github.com/ministryofjustice/cloud-platform-environments.git
cd cloud-platform-environments

# Create a new feature branch for the upgrade
git checkout -b feature/upgrade-postgres-14-to-16-hmpps-community-accommodation-dev

# Navigate to the resources directory
cd namespaces/live.cloud-platform.service.justice.gov.uk/hmpps-community-accommodation-dev/resources
```

#### Step 2.2: Update Terraform Configuration
Edit the `rds.tf` file to include the upgrade configuration:

```hcl
module "rds" {
  source                       = "github.com/ministryofjustice/cloud-platform-terraform-rds-instance?ref=8.1.0"
  db_allocated_storage         = 10
  storage_type                 = "gp2"
  vpc_name                     = var.vpc_name
  team_name                    = var.team_name
  business_unit                = var.business_unit
  application                  = var.application
  is_production                = var.is_production
  environment_name             = var.environment
  infrastructure_support       = var.infrastructure_support
  namespace                    = var.namespace
  performance_insights_enabled = true
  
  # Database upgrade configuration
  db_engine_version            = "16"
  db_instance_class            = "db.t3.small"
  rds_family                   = "postgres16"
  
  # Major version upgrade settings
  allow_major_version_upgrade  = "true"
  prepare_for_major_upgrade    = true
  
  # Additional safety settings for upgrade
  deletion_protection          = false  # Temporarily disable for upgrade
  skip_final_snapshot          = "false"  # Ensure snapshots are created

  providers = {
    aws = aws.london
  }
}
```

#### Step 2.3: Commit and Push Changes
```bash
# Stage the changes
git add rds.tf

# Commit with descriptive message
git commit -m "feat: upgrade PostgreSQL from 14 to 16 for hmpps-community-accommodation-dev

- Update db_engine_version from '14' to '16'
- Update rds_family from 'postgres14' to 'postgres16'
- Enable prepare_for_major_upgrade for safe upgrade
- Temporarily disable deletion_protection for upgrade process
- Ensure skip_final_snapshot is false for backup safety

This change initiates a major version upgrade from PostgreSQL 14 to 16.
The upgrade will be applied through CI/CD pipeline."

# Push the branch to remote
git push origin feature/upgrade-postgres-14-to-16-hmpps-community-accommodation-dev
```

#### Step 2.4: Create Pull Request
1. **Navigate to GitHub**: Go to the cloud-platform-environments repository
2. **Create Pull Request**: 
   - Base branch: `main` (or appropriate target branch)
   - Compare branch: `feature/upgrade-postgres-14-to-16-hmpps-community-accommodation-dev`
3. **Add Description**:
   ```
   ## Database Upgrade: PostgreSQL 14 → 16
   
   ### Changes
   - Upgrade PostgreSQL from version 14 to 16
   - Update RDS family from postgres14 to postgres16
   - Enable prepare_for_major_upgrade for safe upgrade process
   - Temporarily disable deletion_protection during upgrade
   - Ensure final snapshots are created
   
   ### Environment
   - Namespace: hmpps-community-accommodation-dev
   - Environment: Development
   - Risk Level: Low (dev environment)
   
   ### Pre-Upgrade Actions
   - [x] Manual snapshot created
   - [x] Team notified
   - [x] Current state documented
   
   ### Testing Required
   - [ ] Verify application connectivity post-upgrade
   - [ ] Test all application features
   - [ ] Validate database performance
   - [ ] Confirm PostgreSQL 16 version
   
   ### Rollback Plan
   If issues occur, we can restore from the manual snapshot created before upgrade.
   ```

### **Phase 3: CI/CD Deployment**

#### Step 3.1: Pull Request Review
- **Code Review**: Ensure changes are reviewed by team members
- **Terraform Plan**: CI will automatically run `terraform plan` to show changes
- **Validation**: Review the plan output to confirm expected changes:
  - `db_engine_version`: "14" → "16"
  - `rds_family`: "postgres14" → "postgres16"
  - `prepare_for_major_upgrade`: false → true
  - `deletion_protection`: null → false
  - `skip_final_snapshot`: null → "false"

#### Step 3.2: Merge and Deploy
```bash
# After PR approval, merge the pull request
# CI/CD pipeline will automatically:
# 1. Run terraform plan
# 2. Apply terraform changes
# 3. Trigger the database upgrade
```

#### Step 3.3: Monitor CI/CD Progress
- **Monitor Pipeline**: Watch the CI/CD pipeline execution
- **Terraform Apply**: Ensure terraform apply completes successfully
- **Upgrade Initiation**: Confirm RDS upgrade begins

### **Phase 4: Upgrade Monitoring**

#### Step 4.1: Monitor Upgrade Progress
```bash
# Check RDS instance status via Kubernetes
kubectl -n hmpps-community-accommodation-dev get configmap rds-postgresql-instance-output -o yaml

# Monitor AWS RDS console for upgrade progress
# Look for: "Upgrading" → "Available" status
# Expected duration: 30-60 minutes
```

#### Step 4.2: Real-time Monitoring
- **AWS Console**: Monitor RDS instance status
- **Application Logs**: Check for any connectivity issues
- **Team Communication**: Keep team informed of progress

### **Phase 5: Post-Upgrade Validation**

#### Step 5.1: Verify Database Version
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

#### Step 5.2: Application Testing
```bash
# Test application connectivity
# Verify all application features work correctly
# Check for any compatibility issues
```

#### Step 5.3: Performance Validation
```bash
# Monitor database performance
# Check for any performance regressions
# Verify all queries execute correctly
```

### **Phase 6: Post-Upgrade Cleanup**

#### Step 6.1: Create Cleanup Branch
```bash
# Create a new branch for cleanup changes
git checkout -b feature/cleanup-postgres-16-upgrade-hmpps-community-accommodation-dev

# Navigate to the resources directory
cd namespaces/live.cloud-platform.service.justice.gov.uk/hmpps-community-accommodation-dev/resources
```

#### Step 6.2: Update Configuration for Production Readiness
Edit `rds.tf` to remove upgrade-specific settings:

```hcl
module "rds" {
  source                       = "github.com/ministryofjustice/cloud-platform-terraform-rds-instance?ref=8.1.0"
  db_allocated_storage         = 10
  storage_type                 = "gp2"
  vpc_name                     = var.vpc_name
  team_name                    = var.team_name
  business_unit                = var.business_unit
  application                  = var.application
  is_production                = var.is_production
  environment_name             = var.environment
  infrastructure_support       = var.infrastructure_support
  namespace                    = var.namespace
  performance_insights_enabled = true
  
  # Database configuration (PostgreSQL 16)
  db_engine_version            = "16"
  db_instance_class            = "db.t3.small"
  rds_family                   = "postgres16"
  
  # Major version upgrade settings
  allow_major_version_upgrade  = "true"
  prepare_for_major_upgrade    = false  # Disable after successful upgrade
  
  # Safety settings
  deletion_protection          = false  # Keep disabled for dev environment
  skip_final_snapshot          = "false"

  providers = {
    aws = aws.london
  }
}
```

#### Step 6.3: Commit and Deploy Cleanup
```bash
# Stage the changes
git add rds.tf

# Commit with descriptive message
git commit -m "cleanup: remove upgrade-specific settings after PostgreSQL 16 upgrade

- Disable prepare_for_major_upgrade after successful upgrade
- Maintain PostgreSQL 16 configuration
- Keep deletion_protection disabled for dev environment
- Ensure skip_final_snapshot remains false for safety

This cleanup removes temporary upgrade settings while maintaining
the upgraded PostgreSQL 16 configuration."

# Push and create PR for cleanup
git push origin feature/cleanup-postgres-16-upgrade-hmpps-community-accommodation-dev
```

#### Step 6.4: Deploy Cleanup Changes
- Create pull request for cleanup changes
- Review and merge through CI/CD
- Confirm final configuration is applied

## 🔄 Rollback Plan

### **If Upgrade Fails During CI/CD:**

#### Option 1: Restore from Snapshot (Recommended)
```bash
# Create new branch for rollback
git checkout -b hotfix/rollback-postgres-14-hmpps-community-accommodation-dev

# Update rds.tf to restore from snapshot
module "rds" {
  # ... existing configuration ...
  
  # Restore from snapshot
  snapshot_identifier = "arn:aws:rds:region:account:snapshot:snapshot-name"
  
  # Revert to PostgreSQL 14
  db_engine_version = "14"
  rds_family        = "postgres14"
  prepare_for_major_upgrade = false
}
```

#### Option 2: Revert Git Changes
```bash
# Revert the commit that initiated the upgrade
git revert <commit-hash>

# Push revert commit
git push origin main
```

## 📊 Monitoring and Validation Checklist

### **Pre-Upgrade:**
- [ ] Manual snapshot created
- [ ] Team notified
- [ ] Application state documented
- [ ] Feature branch created
- [ ] Terraform changes committed

### **During CI/CD:**
- [ ] Pull request created and reviewed
- [ ] Terraform plan reviewed
- [ ] Changes merged to main
- [ ] CI/CD pipeline executed successfully
- [ ] Upgrade progress monitored

### **Post-Upgrade:**
- [ ] Database version verified (PostgreSQL 16)
- [ ] Application functionality tested
- [ ] Performance validated
- [ ] Cleanup changes deployed
- [ ] Documentation updated

## 🚨 Important Notes

### **CI/CD Considerations:**
- **No Manual Terraform**: All changes go through CI/CD pipeline
- **Pull Request Required**: All changes must be reviewed
- **Automated Testing**: CI runs terraform plan automatically
- **Rollback Process**: Use Git revert or snapshot restoration

### **Timing Considerations:**
- **Upgrade Duration**: 30-60 minutes (major version upgrade)
- **Downtime**: Minimal (RDS handles this automatically)
- **Best Time**: During low-usage hours
- **CI/CD Time**: Additional 10-15 minutes for pipeline execution

### **Risk Mitigation:**
- Development environment allows for safe testing
- Manual snapshot provides rollback option
- `allow_major_version_upgrade = "true"` already enabled
- Performance insights enabled for monitoring
- Git workflow provides change tracking and rollback capability

### **Production Considerations:**
When applying this to production:
1. **Schedule during maintenance window**
2. **Enable deletion_protection = true**
3. **Create multiple backup snapshots**
4. **Test rollback procedures**
5. **Monitor application performance closely**
6. **Have team on standby for issues**
7. **Follow same Git workflow with additional approvals**

## 📝 Documentation Updates

After successful upgrade:
1. Update team documentation
2. Document lessons learned
3. Update production upgrade plan
4. Share findings with other teams
5. Update any runbooks or procedures

This plan follows the organizational workflow where all infrastructure changes are managed through Git and applied via CI/CD, ensuring proper review, testing, and deployment processes.
