# DevSecOps Portfolio Infrastructure

Production-ready AWS infrastructure implementing security best practices, compliance standards, and high availability for a modern portfolio application.

## 🏗️ Infrastructure Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        AWS Infrastructure Architecture                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────────┐  │
│  │      WAF        │    │   CloudFront    │    │         S3 Bucket          │  │
│  │   Protection    │───▶│   Distribution  │───▶│      (Website Host)        │  │
│  │                 │    │                 │    │                             │  │
│  │ • DDoS Shield   │    │ • SSL/TLS       │    │ • Private Access (OAC)     │  │
│  │ • Rate Limiting │    │ • Edge Caching  │    │ • Versioning Enabled       │  │
│  │ • Geo Blocking  │    │ • Compression   │    │ • Encryption at Rest       │  │
│  │ • SQL Injection │    │ • Security Hdrs │    │ • Access Logging           │  │
│  └─────────────────┘    └─────────────────┘    └─────────────────────────────┘  │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │                          Monitoring & Logging Layer                        │  │
│  │                                                                             │  │
│  │ ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │  │
│  │ │ CloudWatch  │  │ CloudTrail  │  │   S3 Logs   │  │    Cost Explorer    │ │  │
│  │ │   Metrics   │  │   Auditing  │  │   Storage   │  │    Optimization     │ │  │
│  │ └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 📁 Infrastructure Components

### Core Resources

#### 1. S3 Website Bucket (`WebsiteBucket`)
- **Purpose**: Hosts the static Next.js portfolio application
- **Security**: Private bucket with Origin Access Control (OAC)
- **Features**: Versioning, encryption, access logging
- **Backup**: Automated backup to separate bucket

#### 2. CloudFront Distribution (`CloudFrontDistribution`)
- **Purpose**: Global content delivery with edge caching
- **Security**: WAF integration, SSL/TLS termination, security headers
- **Performance**: HTTP/2, Gzip compression, optimized caching
- **Monitoring**: Real-time logs and metrics

#### 3. WAF Web ACL (Imported from WAF stack)
- **Purpose**: Application-layer security protection
- **Features**: DDoS protection, rate limiting, geo-blocking
- **Rules**: OWASP Top 10 protection, custom security rules
- **Monitoring**: Security event logging and alerting

#### 4. Logging Infrastructure (`LoggingBucket`)
- **Purpose**: Centralized logging for all services
- **Storage**: S3 with lifecycle policies for cost optimization
- **Access**: Restricted access with proper IAM policies
- **Retention**: Configurable retention periods

## 🚀 Quick Deployment (10 Minutes)

### Prerequisites
- AWS CLI configured with appropriate permissions
- CloudFormation deployment permissions
- WAF stack deployed (prerequisite)

### 1. Deploy WAF Stack (Security Layer)
```bash
# Deploy WAF protection first
aws cloudformation deploy \
  --template-file waf-stack.yaml \
  --stack-name devsecops-portfolio-waf \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    ProjectName=devsecops-portfolio

# Verify WAF deployment
aws cloudformation describe-stacks \
  --stack-name devsecops-portfolio-waf \
  --query 'Stacks[0].StackStatus'
```

### 2. Deploy Main Infrastructure
```bash
# Deploy core infrastructure
aws cloudformation deploy \
  --template-file portfolio-infrastructure.yaml \
  --stack-name devsecops-portfolio-infrastructure \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    ProjectName=devsecops-portfolio \
    LoggingBucketName=devsecops-portfolio-logs

# Get deployment outputs
aws cloudformation describe-stacks \
  --stack-name devsecops-portfolio-infrastructure \
  --query 'Stacks[0].Outputs'
```

### 3. Verify Deployment
```bash
# Test website accessibility
WEBSITE_URL=$(aws cloudformation describe-stacks \
  --stack-name devsecops-portfolio-infrastructure \
  --query 'Stacks[0].Outputs[?OutputKey==`WebsiteURL`].OutputValue' \
  --output text)

curl -I $WEBSITE_URL

# Check security headers
curl -I $WEBSITE_URL | grep -E "(Strict-Transport-Security|Content-Security-Policy|X-Frame-Options)"
```

## 🛡️ Security Implementation

### Network Security

#### CloudFront Security Headers
```yaml
ResponseHeadersPolicy:
  Type: AWS::CloudFront::ResponseHeadersPolicy
  Properties:
    ResponseHeadersPolicyConfig:
      SecurityHeadersConfig:
        StrictTransportSecurity:
          AccessControlMaxAgeSec: 31536000
          IncludeSubdomains: true
        ContentTypeOptions:
          Override: true
        FrameOptions:
          FrameOption: DENY
          Override: true
        ReferrerPolicy:
          ReferrerPolicy: strict-origin-when-cross-origin
          Override: true
```

#### WAF Rules Configuration
```bash
# View WAF rules
aws wafv2 describe-web-acl \
  --scope CLOUDFRONT \
  --id $(aws cloudformation describe-stacks \
    --stack-name devsecops-portfolio-waf \
    --query 'Stacks[0].Outputs[?OutputKey==`WebACLId`].OutputValue' \
    --output text)

# Monitor WAF metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/WAFV2 \
  --metric-name BlockedRequests \
  --dimensions Name=WebACL,Value=devsecops-portfolio-waf \
  --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum
```

### Data Security

#### S3 Bucket Security
- **Encryption**: AES-256 server-side encryption
- **Access Control**: Bucket policies with least privilege
- **Public Access**: Completely blocked, access only via CloudFront
- **Versioning**: Enabled for data protection and rollback
- **MFA Delete**: Optional for additional protection

#### Origin Access Control (OAC)
```yaml
OriginAccessControl:
  Type: AWS::CloudFront::OriginAccessControl
  Properties:
    OriginAccessControlConfig:
      Name: !Sub ${ProjectName}-oac
      OriginAccessControlOriginType: s3
      SigningBehavior: always
      SigningProtocol: sigv4
```

### Compliance Features

#### Audit Logging
- **CloudTrail**: API call logging and monitoring
- **S3 Access Logs**: Detailed access patterns
- **CloudFront Logs**: Request-level logging
- **WAF Logs**: Security event logging

#### Data Retention
```yaml
LoggingBucketLifecycle:
  Type: AWS::S3::BucketPolicy
  Properties:
    LifecycleConfiguration:
      Rules:
        - Id: LogRetention
          Status: Enabled
          Transitions:
            - TransitionInDays: 30
              StorageClass: STANDARD_IA
            - TransitionInDays: 90
              StorageClass: GLACIER
          ExpirationInDays: 2555  # 7 years for compliance
```

## 📊 Monitoring & Alerting

### CloudWatch Dashboards

#### Infrastructure Health Dashboard
```bash
# Create infrastructure monitoring dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "DevSecOps-Infrastructure-Health" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/CloudFront", "Requests", "DistributionId", "'$DISTRIBUTION_ID'"],
            ["AWS/CloudFront", "BytesDownloaded", "DistributionId", "'$DISTRIBUTION_ID'"],
            ["AWS/CloudFront", "4xxErrorRate", "DistributionId", "'$DISTRIBUTION_ID'"],
            ["AWS/CloudFront", "5xxErrorRate", "DistributionId", "'$DISTRIBUTION_ID'"]
          ],
          "period": 300,
          "stat": "Sum",
          "region": "us-east-1",
          "title": "CloudFront Performance Metrics"
        }
      }
    ]
  }'
```

#### Security Monitoring Dashboard
```bash
# WAF security metrics
aws cloudwatch put-dashboard \
  --dashboard-name "DevSecOps-Security-Metrics" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/WAFV2", "AllowedRequests", "WebACL", "devsecops-portfolio-waf"],
            ["AWS/WAFV2", "BlockedRequests", "WebACL", "devsecops-portfolio-waf"],
            ["AWS/WAFV2", "CountedRequests", "WebACL", "devsecops-portfolio-waf"]
          ],
          "period": 300,
          "stat": "Sum",
          "region": "us-east-1",
          "title": "WAF Security Events"
        }
      }
    ]
  }'
```

### Automated Alerts

#### Critical Infrastructure Alerts
```yaml
HighErrorRateAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub ${ProjectName}-high-error-rate
    AlarmDescription: High 4xx/5xx error rate detected
    MetricName: 4xxErrorRate
    Namespace: AWS/CloudFront
    Statistic: Average
    Period: 300
    EvaluationPeriods: 2
    Threshold: 5
    ComparisonOperator: GreaterThanThreshold
    Dimensions:
      - Name: DistributionId
        Value: !Ref CloudFrontDistribution
    AlarmActions:
      - !Ref SNSTopicArn
```

#### Security Alerts
```yaml
SecurityIncidentAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub ${ProjectName}-security-incident
    AlarmDescription: High number of blocked requests detected
    MetricName: BlockedRequests
    Namespace: AWS/WAFV2
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 1
    Threshold: 100
    ComparisonOperator: GreaterThanThreshold
    Dimensions:
      - Name: WebACL
        Value: !ImportValue
          Fn::Sub: '${ProjectName}-waf-name'
```

## 🔧 Configuration Management

### Environment-Specific Parameters

#### Development Environment
```bash
# Deploy with development settings
aws cloudformation deploy \
  --template-file portfolio-infrastructure.yaml \
  --stack-name devsecops-portfolio-dev \
  --parameter-overrides \
    ProjectName=devsecops-portfolio-dev \
    Environment=development \
    LogRetentionDays=30 \
    CachingEnabled=false
```

#### Production Environment
```bash
# Deploy with production settings
aws cloudformation deploy \
  --template-file portfolio-infrastructure.yaml \
  --stack-name devsecops-portfolio-prod \
  --parameter-overrides \
    ProjectName=devsecops-portfolio-prod \
    Environment=production \
    LogRetentionDays=2555 \
    CachingEnabled=true \
    WAFEnabled=true
```

### Custom Domain Configuration

#### Route 53 Setup
```yaml
# Add to template for custom domain
HostedZone:
  Type: AWS::Route53::HostedZone
  Properties:
    Name: !Ref DomainName

AliasRecord:
  Type: AWS::Route53::RecordSet
  Properties:
    HostedZoneId: !Ref HostedZone
    Name: !Ref DomainName
    Type: A
    AliasTarget:
      DNSName: !GetAtt CloudFrontDistribution.DomainName
      HostedZoneId: Z2FDTNDATAQYW2  # CloudFront hosted zone ID
```

#### SSL Certificate
```yaml
SSLCertificate:
  Type: AWS::CertificateManager::Certificate
  Properties:
    DomainName: !Ref DomainName
    SubjectAlternativeNames:
      - !Sub www.${DomainName}
    ValidationMethod: DNS
    DomainValidationOptions:
      - DomainName: !Ref DomainName
        HostedZoneId: !Ref HostedZone
```

## 🔄 Backup & Recovery

### Automated Backup Strategy

#### S3 Cross-Region Replication
```yaml
ReplicationConfiguration:
  Type: AWS::S3::Bucket
  Properties:
    ReplicationConfiguration:
      Role: !GetAtt ReplicationRole.Arn
      Rules:
        - Id: ReplicateToSecondaryRegion
          Status: Enabled
          Prefix: ''
          Destination:
            Bucket: !Sub arn:aws:s3:::${ProjectName}-backup-${AWS::Region}
            StorageClass: STANDARD_IA
```

#### Point-in-Time Recovery
```bash
#!/bin/bash
# Restore website to specific point in time

BUCKET_NAME="devsecops-portfolio-website"
RESTORE_DATE="2024-01-15T10:30:00Z"

# List versions around restore date
aws s3api list-object-versions \
  --bucket $BUCKET_NAME \
  --prefix "index.html" \
  --query "Versions[?LastModified<='$RESTORE_DATE']|[0]"

# Restore specific version
aws s3api copy-object \
  --copy-source "$BUCKET_NAME/index.html?versionId=$VERSION_ID" \
  --bucket $BUCKET_NAME \
  --key "index.html"
```

### Disaster Recovery Procedures

#### Infrastructure Recovery
```bash
#!/bin/bash
# Complete infrastructure recovery script

echo "🚨 Starting disaster recovery procedure..."

# 1. Deploy infrastructure in secondary region
aws cloudformation deploy \
  --template-file portfolio-infrastructure.yaml \
  --stack-name devsecops-portfolio-dr \
  --region us-west-2 \
  --parameter-overrides \
    ProjectName=devsecops-portfolio-dr \
    Environment=disaster-recovery

# 2. Restore data from backup
aws s3 sync s3://devsecops-portfolio-backup-us-east-1 s3://devsecops-portfolio-dr-website --region us-west-2

# 3. Update DNS to point to DR environment
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch file://dns-failover.json

echo "✅ Disaster recovery completed"
```

## 🚨 Troubleshooting Guide

### Common Issues

#### CloudFront Distribution Issues

**Issue**: 403 Forbidden errors
```bash
# Check OAC configuration
aws cloudfront get-origin-access-control --id $OAC_ID

# Verify S3 bucket policy
aws s3api get-bucket-policy --bucket devsecops-portfolio-website

# Test direct S3 access (should fail)
curl -I https://devsecops-portfolio-website.s3.amazonaws.com/index.html
```

**Issue**: Slow cache invalidation
```bash
# Check invalidation status
aws cloudfront list-invalidations --distribution-id $DISTRIBUTION_ID

# Create targeted invalidation
aws cloudfront create-invalidation \
  --distribution-id $DISTRIBUTION_ID \
  --paths "/index.html" "/static/css/*"
```

#### S3 Access Issues

**Issue**: Upload failures
```bash
# Check bucket permissions
aws s3api get-bucket-acl --bucket devsecops-portfolio-website

# Test upload with debug
aws s3 cp test.html s3://devsecops-portfolio-website/ --debug

# Verify IAM permissions
aws iam simulate-principal-policy \
  --policy-source-arn $ROLE_ARN \
  --action-names s3:PutObject \
  --resource-arns arn:aws:s3:::devsecops-portfolio-website/*
```

#### WAF Blocking Issues

**Issue**: Legitimate traffic blocked
```bash
# Check WAF logs
aws logs filter-log-events \
  --log-group-name aws-waf-logs-devsecops-portfolio \
  --start-time $(date -d '1 hour ago' +%s)000 \
  --filter-pattern '{ $.action = "BLOCK" }'

# Temporarily disable specific rule
aws wafv2 update-rule-group \
  --scope CLOUDFRONT \
  --id $RULE_GROUP_ID \
  --rules file://updated-rules.json
```

### Performance Optimization

#### CloudFront Optimization
```yaml
# Optimized cache behaviors
CacheBehaviors:
  - PathPattern: "/static/*"
    TargetOriginId: S3Origin
    ViewerProtocolPolicy: redirect-to-https
    CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad  # CachingOptimizedForUncompressedObjects
    Compress: true
    
  - PathPattern: "/api/*"
    TargetOriginId: S3Origin
    ViewerProtocolPolicy: redirect-to-https
    CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad  # CachingDisabled
    OriginRequestPolicyId: 88a5eaf4-2fd4-4709-b370-b4c650ea3fcf  # CORS-S3Origin
```

#### S3 Performance Tuning
```bash
# Enable transfer acceleration
aws s3api put-bucket-accelerate-configuration \
  --bucket devsecops-portfolio-website \
  --accelerate-configuration Status=Enabled

# Optimize multipart upload threshold
aws configure set default.s3.multipart_threshold 64MB
aws configure set default.s3.multipart_chunksize 16MB
```

## 💰 Cost Optimization

### Cost Monitoring

#### Budget Alerts
```yaml
InfrastructureBudget:
  Type: AWS::Budgets::Budget
  Properties:
    Budget:
      BudgetName: !Sub ${ProjectName}-infrastructure-budget
      BudgetLimit:
        Amount: 50
        Unit: USD
      TimeUnit: MONTHLY
      BudgetType: COST
      CostFilters:
        TagKey:
          - Project
        TagValue:
          - !Ref ProjectName
    NotificationsWithSubscribers:
      - Notification:
          NotificationType: ACTUAL
          ComparisonOperator: GREATER_THAN
          Threshold: 80
        Subscribers:
          - SubscriptionType: EMAIL
            Address: admin@example.com
```

#### Cost Analysis
```bash
# Get monthly costs by service
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter file://cost-filter.json

# Analyze CloudFront costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity DAILY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=USAGE_TYPE \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon CloudFront"]}}'
```

### Optimization Strategies

#### S3 Storage Classes
```yaml
LifecycleConfiguration:
  Rules:
    - Id: OptimizeStorageCosts
      Status: Enabled
      Transitions:
        - TransitionInDays: 30
          StorageClass: STANDARD_IA
        - TransitionInDays: 90
          StorageClass: GLACIER
        - TransitionInDays: 365
          StorageClass: DEEP_ARCHIVE
```

#### CloudFront Price Class
```yaml
# Use PriceClass_100 for cost optimization (US, Canada, Europe)
PriceClass: PriceClass_100

# Or PriceClass_200 for better global performance
# PriceClass: PriceClass_200
```

## 🎯 Next Steps

### Immediate Actions (Next Hour)
1. **Deploy Infrastructure**: Run CloudFormation deployment
2. **Verify Security**: Test WAF rules and security headers
3. **Configure Monitoring**: Set up CloudWatch dashboards
4. **Test Performance**: Run load tests and optimize caching
5. **Document Access**: Update team documentation

### Short-term Enhancements (Next Week)
1. **Custom Domain**: Configure Route 53 and SSL certificate
2. **Advanced WAF**: Implement custom security rules
3. **Multi-Region**: Set up disaster recovery in secondary region
4. **Cost Optimization**: Implement lifecycle policies and budgets
5. **Compliance**: Configure additional audit logging

### Long-term Improvements (Next Month)
1. **Infrastructure as Code**: Implement Terraform or CDK
2. **GitOps**: Integrate with CI/CD pipelines
3. **Advanced Monitoring**: Implement distributed tracing
4. **Security Automation**: Automated threat response
5. **Performance**: Implement edge computing with Lambda@Edge

## 📞 Support

For infrastructure issues:
1. Check CloudFormation stack events
2. Review CloudWatch logs and metrics
3. Verify IAM permissions and policies
4. Test connectivity and DNS resolution
5. Contact AWS Support for service-specific issues

## 📄 License

This infrastructure code is licensed under the MIT License - see the LICENSE file for details.