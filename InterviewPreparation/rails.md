# Ruby on Rails Master Key Guide

## What is the Rails Master Key?

The Rails master key is a single, secure key that encrypts and decrypts all encrypted credentials in your Rails application. It's stored in the `config/master.key` file and is used to decrypt the `config/credentials.yml.enc` file.

## How Rails Master Key Works

### 1. **Encryption Process**
- Rails uses the master key to encrypt sensitive data in `credentials.yml.enc`
- The master key itself is never committed to version control
- Only the encrypted credentials file is committed

### 2. **File Structure**
```
config/
├── master.key          # Contains the master key (NEVER commit this)
├── credentials.yml.enc # Encrypted credentials (safe to commit)
└── credentials.yml     # Plain text credentials (NEVER commit this)
```

### 3. **Usage in Code**
```ruby
# Access encrypted credentials
Rails.application.credentials.database_url
Rails.application.credentials.secret_key_base
Rails.application.credentials.aws[:access_key_id]
```

## Benefits of Rails Master Key

### 1. **Security**
- **Centralized Encryption**: Single key protects all sensitive data
- **No Plain Text in Repo**: Sensitive data is never stored in plain text
- **Environment Isolation**: Different keys for different environments

### 2. **Simplified Management**
- **Single Key**: One key to manage instead of multiple environment variables
- **Automatic Decryption**: Rails handles decryption automatically
- **Version Control Safe**: Encrypted files can be safely committed

### 3. **Environment Flexibility**
```ruby
# config/environments/production.rb
config.require_master_key = true

# config/environments/development.rb
config.require_master_key = false
```

### 4. **Deployment Benefits**
- **Docker**: Single environment variable for the master key
- **Kubernetes**: One secret instead of multiple
- **CI/CD**: Simplified secret management

## Rails Master Key vs Helm Charts

### **Rails Master Key Approach**

#### Advantages:
- **Application-Level Security**: Encryption happens at the application level
- **Language-Native**: Built into Rails framework
- **Simple Setup**: Minimal configuration required
- **Version Control Safe**: Encrypted files can be committed
- **Environment Agnostic**: Works the same across all deployment methods

#### Disadvantages:
- **Rails-Specific**: Only works with Rails applications
- **Single Point of Failure**: One key protects everything
- **Limited Granularity**: All-or-nothing encryption

### **Helm Charts Approach**

#### Advantages:
- **Kubernetes-Native**: Designed specifically for Kubernetes
- **Granular Control**: Different secrets for different components
- **Infrastructure-Level**: Manages secrets at the infrastructure level
- **Multi-Language Support**: Works with any application
- **Advanced Features**: Secret rotation, RBAC, etc.

#### Disadvantages:
- **Complexity**: Requires Kubernetes knowledge
- **Infrastructure Coupling**: Tied to Kubernetes deployment
- **Overhead**: Additional complexity for simple applications
- **Learning Curve**: Steeper learning curve for developers

## When to Use Each Approach

### **Use Rails Master Key When:**
- ✅ Building a Rails-only application
- ✅ Simple deployment requirements
- ✅ Small to medium team
- ✅ Quick development and deployment
- ✅ Non-Kubernetes environments

### **Use Helm Charts When:**
- ✅ Deploying to Kubernetes
- ✅ Complex microservices architecture
- ✅ Multiple applications sharing secrets
- ✅ Advanced secret management requirements
- ✅ Large team with DevOps expertise

## Best Practices

### 1. **Master Key Management**
```bash
# Generate new master key
rails credentials:edit

# View current credentials
rails credentials:show

# Add new credentials
rails credentials:edit
```

### 2. **Environment Variables**
```bash
# Set master key in production
export RAILS_MASTER_KEY=your_master_key_here

# Docker
docker run -e RAILS_MASTER_KEY=your_key your_app

# Kubernetes
kubectl create secret generic rails-master-key \
  --from-literal=master-key=your_key
```

### 3. **Security Considerations**
- Never commit `master.key` to version control
- Use different keys for different environments
- Rotate keys regularly in production
- Limit access to master key files

### 4. **Hybrid Approach**
For complex deployments, you can use both:
- Rails master key for application-level secrets
- Helm charts for infrastructure-level secrets
- Kubernetes secrets for the master key itself

## Example Implementation

### **Simple Rails App with Master Key**
```ruby
# config/credentials.yml.enc (encrypted)
database_url: postgresql://user:pass@localhost/db
secret_key_base: [long_random_string]
aws:
  access_key_id: [encrypted_key]
  secret_access_key: [encrypted_secret]
```

### **Kubernetes Deployment with Helm**
```yaml
# values.yaml
rails:
  masterKey: "{{ .Values.rails.masterKey }}"
  environment: production

# templates/deployment.yaml
env:
  - name: RAILS_MASTER_KEY
    valueFrom:
      secretKeyRef:
        name: rails-secrets
        key: master-key
```

## Conclusion

The Rails master key provides a simple, secure way to manage application secrets, especially for Rails applications. While Helm charts offer more advanced features for Kubernetes deployments, the master key approach is often sufficient and simpler for many use cases.

Choose based on your specific needs:
- **Rails Master Key**: Simple, secure, Rails-native
- **Helm Charts**: Advanced, Kubernetes-native, complex deployments
- **Hybrid**: Best of both worlds for complex scenarios
