# Daily Workflow Guide

## How to Use This Implementation Guide

Each day is designed to be completed in 2-4 hours with specific, actionable tasks. This document explains how to work through each day effectively.

## Daily Structure

### Standard Day Format

Each day's guide contains:

```markdown
### Day X: [Title]
**Time: X-Y hours**

#### Objectives
- Clear goals for the day
- Expected outcomes

#### Prerequisites  
- What must be completed first
- Dependencies from previous days

#### Tasks
1. **Task 1 (time estimate)**
   - Step-by-step instructions
   - Exact commands to run
   - Code snippets to implement

2. **Task 2 (time estimate)**
   - [Similar structure]

#### Validation
- Commands to verify your work
- Expected outputs
- How to know you're done

#### Next Day Setup
- Preparation for tomorrow
- Files to review
- Concepts to understand
```

## Working Through Each Day

### 🌅 Start of Day Routine (10 minutes)

1. **Review Previous Work**
   ```bash
   # Check git status
   git status
   
   # Review what you built yesterday
   git log --oneline -5
   
   # Check if services are running
   docker-compose ps
   ```

2. **Read Today's Objectives**
   - Understand what you'll accomplish
   - Note the time estimate
   - Identify any learning areas

3. **Set Up Your Environment**
   ```bash
   # Start services if needed
   docker-compose up -d
   
   # Pull latest changes (if working with a team)
   git pull origin main
   
   # Check Node.js is available
   node --version
   ```

### 🔨 During Development (Main Work)

#### Follow the Task Structure

1. **Read Each Task Completely First**
   - Don't start coding immediately
   - Understand the full task scope
   - Note file locations and commands

2. **Execute Commands Exactly as Written**
   ```bash
   # Copy commands exactly - they're tested
   # Example: DO copy this exactly
   mkdir -p packages/{contracts,backend,frontend,sdk}
   
   # Don't modify unless you understand implications
   ```

3. **Use Code Snippets as Templates**
   - Copy the provided code first
   - Modify only if you understand the changes
   - Test frequently as you build

4. **Test as You Go**
   ```bash
   # Run validations after each major step
   npm run test
   npm run lint
   docker-compose ps
   ```

#### Git Workflow

Use consistent git practices:

```bash
# Start each task with a clean state
git status

# Make commits for each completed task
git add .
git commit -m "feat: implement [task description]"

# Push at end of day
git push origin feature/day-X-[description]
```

### 🧪 Validation Process

#### After Each Task
```bash
# Check if your code compiles
npm run build

# Run relevant tests
npm run test -- [specific-test-pattern]

# Verify services are healthy
docker-compose ps
curl http://localhost:3000/health
```

#### End of Day Validation
Each day has specific validation steps:

```bash
# Example validation commands
npm run lint           # Code quality
npm run type-check     # TypeScript validation
npm run test           # All tests pass
docker-compose logs    # Check for errors
```

**✅ Only mark a day complete when ALL validations pass**

### 🌙 End of Day Routine (15 minutes)

1. **Run Full Validation**
   - Execute all validation commands
   - Fix any failing tests
   - Ensure clean git status

2. **Document Progress**
   ```bash
   # Create a daily log entry
   echo "## Day X - $(date)" >> DAILY_LOG.md
   echo "✅ Completed: [list accomplishments]" >> DAILY_LOG.md
   echo "📝 Notes: [any important notes]" >> DAILY_LOG.md
   echo "🔄 Next: [tomorrow's focus]" >> DAILY_LOG.md
   echo "" >> DAILY_LOG.md
   ```

3. **Prepare for Tomorrow**
   - Read next day's objectives
   - Note any prerequisites
   - Plan your schedule

4. **Clean Shutdown**
   ```bash
   # Commit final work
   git add .
   git commit -m "feat: complete day X implementation"
   
   # Shutdown services to save resources
   docker-compose down
   
   # Push to remote
   git push origin your-branch
   ```

## Time Management

### If You're Running Behind

**Option 1: Focus on Core Features**
- Skip optional enhancements
- Implement minimum viable version
- Note features to add later

**Option 2: Extend the Day**
- Some days can be split across 2 sessions
- Maintain momentum on complex tasks
- Don't break in the middle of a logical unit

**Option 3: Get Help**
- Check troubleshooting guide
- Review error messages carefully
- Ask for help with specific code blocks

### If You're Ahead of Schedule

**Option 1: Add Optional Features**
- Implement "nice-to-have" features
- Add extra tests
- Improve documentation

**Option 2: Prepare for Future Days**
- Read ahead in the guide
- Set up additional tools
- Research upcoming technologies

**Option 3: Optimize Current Code**
- Refactor for better performance
- Add error handling
- Improve code organization

## Common Patterns

### Working with Packages
```bash
# Always work from correct directory
cd packages/[package-name]

# Use package-specific commands
npm run build
npm run test
npm run dev

# Return to root for workspace commands
cd ../..
npm run build  # Builds all packages
```

### Database Operations
```bash
# Start database first
docker-compose up -d postgres redis

# Run migrations
npm run migration:run

# Check database connection
docker exec -it [postgres-container] psql -U loyalty_user -d loyalty_platform
```

### Smart Contract Development
```bash
# Always from contracts directory
cd packages/contracts

# Compile after changes
npx hardhat compile

# Run tests frequently
npx hardhat test

# Deploy to testnet when ready
npx hardhat run scripts/deploy.ts --network polygon_mumbai
```

## Troubleshooting Workflow

### When Things Don't Work

1. **Check the Basics**
   ```bash
   # Are services running?
   docker-compose ps
   
   # Are there error messages?
   docker-compose logs
   
   # Is your code syntactically correct?
   npm run lint
   ```

2. **Compare with Expected State**
   - Re-read the task instructions
   - Check if you missed a step
   - Verify file contents match examples

3. **Isolate the Problem**
   ```bash
   # Test individual components
   npm run test -- specific-test
   
   # Check network connectivity
   curl http://localhost:3000
   
   # Verify environment variables
   echo $DATABASE_URL
   ```

4. **Get Back to Working State**
   ```bash
   # Reset to last working commit
   git stash
   git reset --hard HEAD
   
   # Or start the failing task over
   git checkout -- problematic-file
   ```

## Quality Standards

### Code Quality Checklist
- [ ] Code compiles without errors
- [ ] All tests pass
- [ ] Linting rules satisfied
- [ ] No console errors in browser/terminal
- [ ] Environment variables properly set
- [ ] Git commits are clean and descriptive

### Documentation Standards
- [ ] README files updated
- [ ] Code comments for complex logic
- [ ] API documentation current
- [ ] Environment setup documented

### Security Checklist
- [ ] No hardcoded secrets
- [ ] Proper input validation
- [ ] Error messages don't leak sensitive info
- [ ] Access controls implemented correctly

---

## Ready to Start?

1. **Choose your starting week**: Go to `week-by-week/week-01/`
2. **Read the week overview**: Understand the goals
3. **Start with Day 1**: Follow the daily structure
4. **Maintain momentum**: Work consistently every day

**Remember**: The guide is designed to be followed sequentially. Each day builds on the previous day's work, so don't skip ahead unless you're certain about dependencies.