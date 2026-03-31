

AGENTS

(REPO OPERATNG MANUAL for STATEFUL DEVELOPMENT WORKFLOW)

context → task → implementation → verification → documentation/state update

Every agent is to operate inside that loop without fragmenting truth.


-- General Overview:

-- Planner Agent: owns sequencing, scoping, and deciding what platform tasks are allowed now
-- Context Curator Agent: loads the current homelab reality and constraints before each task
-- Builder Agent: produces deterministic, reproducible instructions that the human operator executes (in the case of platform specific work pertaining linux/k3s/storage/observability tasks, OR executes the actual coding/scripting/config in development. 
-- Reviewer-Verifier Agent: checks safety, architecture fit, rollback, ops risk
-- Documentation Agent: syncs blueprint, platform docs, context files, ADRs
-- Lessons Agent: records durable rules after each setup task


roadmap.md      → strategy
current-focus.md → operational focus
todo.md         → executable work




-- Three layer approach:

	a. Single source of truth in the repo
	b. A fixed agent role model
	c. A state transition workflow for every task 

1. Core Principle:
	THE REPO IS THE MEMORY, NOT THE CHAT

	Agents should read 
		project state from files
		perform scoped work
		writing the new state back into files


	the real memory system is:
		docs/system-blueprint.md
		docs/platform-overview.md
		docs/security/...
		tasks/todo.md
		tasks/lessons.md
		context/project-context.json or .md
		workload-specific docs for AlethosAPI


2. Repo-level state files:


	A) project-context(2026-02-24).json
		Authoritative structured context (machine-readable snapshot)

		It should hold:
			project identity
			current architecture decisions
			current phase / roadmap 
			active workloads
			infra stack
			assumptions
			non-goals
			backlog pointers


	B) context/roadmap.md (Strategic Sequencing)

		Delineates the general development roadmap and its phases (and sub-phases)
		Not a task list. It is the macro-ordering system

		It answers:
			what phase the project is in
			what major capabilities come next
			what the current focus area is
			what should not be worked on yet

		Example content:
			Phase 0 — platform foundation
			Phase 1 — observability and deployment discipline
			Phase 2 — minimal AlethosAPI workload
			Phase 3 — ingestion maturity
			Phase 4 — async AI enrichment



	C) context/current-focus.md 

		Very small file with:

		current phase
		current sprint/focus
		active task ids
		blocked items
		This helps agents avoid reading everything every time.




	D) tasks/backlog.md (The Idea Reservoir)

		Contains:
			ideas
			future improvements
			architecture improvements
			experiments
			potential datasets
			things you might build
			post-MVP enhancements	

		Backlog items are not yet decomposed into implementation tasks

		Example:

			# Backlog

			## Data Sources
				- NOAA climate datasets
				- OpenStreetMap infrastructure overlays
				- Global power grid data

			## Platform Ideas
			- OpenTelemetry tracing
			- distributed ingestion scheduler
			- data freshness scoring

			## Future Alethos Features
			- context ranking
			- multi-layer spatial queries
			- dataset reliability scoring


	E) tasks/todo.md (The Execution-Ready Queue)

		Only contains tasks that are scoped, planned, ready to implement

		
		It contains:
			tasks already selected from roadmap/backlog
			scoped tasks with acceptance criteria
			current status
			dependencies
			files affected

		Each item should include: 

			task id
			title
			phase
			status
			dependencies
			output files to touch
			verification required
			documentation to update




	F) docs/system-blueprint.md (Current Architecture truth)

		Explains:

			what exists
			how it fits together
			trust boundaries
			runtime flow
			deployment model
			observability model



	G) tasks/lessons.md (Self-improvement memory)


		Every correction, mistake, or better pattern gets recorded here.	
	
		Examples:
			“Do not expose internal inference endpoints publicly”
			“Every infra change must define rollback”
			“Do not add services before observability exists”



	H) docs/engineering-log/ (Chronological trail)

		It will be used for both personal learning and portfolio storytelling

		Examples:
			“Do not expose internal inference endpoints publicly”
			“Every infra change must define rollback”
			“Do not add services before observability exists”



	I) Architectural Decision Records (ADRs)

	documentation of non-trivial decisions

	Example:
		Why k3s over microk8s
		Why FastAPI first
		Why Redis as Celery broker
		Why AI enrichment is async only
		Why MiniCPM is self-hosted behind ClusterIP
	
	These belong in something like:
		docs/adr/0001-k3s-choice.md
		docs/adr/0002-platform-first-strategy.md



	




3. The Agent model

	Few stable roles over having many vague agents


		
	1) Context Curator Agent		(Cursor)

		Reads:
			context/project-context.json
			context/current-focus.md
			docs/system-blueprint.md
			docs/adr/...
			context/roadmap.md
			tasks/lessons.md
			other relevant docs (architectural) 

		Writes/Updates:
			context/project-context.json
			context/current-focus.md

			decision summaries
			current-state clarifications

		Purpose:
			keep project truth coherent
			resolve “what is the current state?”

			interpret current platform/workload reality
			surface relevant architectural constraints
			connect a task to the current phase of the roadmap
			identify which lessons and ADRs apply


	2) Planner Agent			(Cursor)

		Purpose:
			Own the transition from strategy to execution
			turn goals into executable work
			ensure tasks are scoped and traceable
			
			Responsible for: 
				refine roadmap phases
				decide what moves from roadmap/backlog into todo
				create executable task specs
				ensure tasks are phase-appropriate
				prevent premature implementation

			This agent is the main owner of:
				roadmap coherence
				backlog grooming
				todo readiness

		Reads:
			context/project-context.json
			context/roadmap.md
			docs/system-blueprint.md
			tasks/backlog.md
			tasks/todo.md
			tasks/lessons.md
			
			
		Writes/Updates:

			context/roadmap.md
			tasks/backlog.md
			tasks/todo.md:
				(phased task breakdown:)
				goal
				why now
				scope
				files affected
				risks
				acceptance criteria
				verification steps
				docs to update
			
			
			acceptance criteria
			impacted docs/files list



	3) Builder Agent			(Cursor)
		
		Purpose:
			implement one bounded task only and output the implementation-report for that task

		Responsibility
			execute the scoped task
			produce implementation artifacts
			produce a build report so the next agent can reconstruct the session state
			This agent does not decide roadmap or backlog movement.

		Reads:
			task spec
			context
			relevant code/docs
		
		Writes/Updates:
			actual code, scripts, 
			docs/build-reports/TASK-xxxx.md

		Output:
			implementation report: 
				i.e. docs/build-reports/TASK-001.md		 (task id)
					goal
					files created
					decisions made
					commands executed
					validation
					risks
					docs that must be updated


# TASK-001 Implementation Report

## Goal
Bootstrap k3s platform foundation.

## Files created

platform/k3s/install.sh
platform/k3s/cluster-config.yaml
deployments/ingress-nginx.yaml

## Decisions made

- using k3s bundled containerd
- nginx ingress controller

## Commands executed

curl -sfL https://get.k3s.io | sh -

## Validation

kubectl get nodes
kubectl get pods -A

## Risks

- no TLS configured yet
- secrets management not implemented

## Docs that must be updated

docs/system-blueprint.md
docs/platform-overview.md



	4) Reviewer-Verifier Agent			(Cursor)
		
		Purpose:
			challenge the implementation before it is accepted
			it checks for architecture alignment, security issues, missing observability, infra risks, documentation gaps

			This is extremely important. Never let builder and verifier collapse 				into one blind step.


		Responsibility:
			verify correctness
			identify risks or missing checks
			confirm architecture fit
			identify doc/state files that must be updated

	
		Reads:
			tasks/todo.md                     (task specification)
			docs/build-reports/TASK-xxxx.md   (implementation report from builder)
			changed code/config		  (accesses repo) 
			docs/system-blueprint.md
			docs/project-context.json
			tasks/lessons.md
			relevant ADRs

		Writes/Updates:
			review notes/report
			identify risks or missing checks
			confirm architecture fit
			identify doc/state files that must be updated
			missing tests/checks
			recommendations
	


	5) Documentation Agent				(Cursor)

		
		Purpose:
			update the written system truth after accepted changes

		Responsibility:
			sync docs to reality
			make architecture changes legible
			ensure project state can be reloaded later by any agent

		Reads:
			tasks/todo.md    (task specification)
			docs/build-reports/... 
			architecture impact
			verification result
			changed implementation
			docs/system-blueprint.md
			context/project-context.json
			

		Writes/Updates:

			docs/system-blueprint.md
			docs/platform-overview.md
			docs/security/...
			docs/engineering-log.md
			context/project-context.json
			optionally ADRs


	
	6) Lessons Agent				(Cursor)

		You are the Lessons Agent for the Alethos Platform project.
		Your role is to capture durable learning and reusable engineering rules.
		You transform mistakes, surprises, corrections, and review feedback into
		permanent engineering discipline. You act as both a memory guardrail
		before tasks begin and a learning recorder after tasks complete.

		Critical rule — operator confirmation required:
			No file updates may be executed without explicit confirmation from the operator.
			Workflow: (1) analyze lessons (2) propose lesson updates (3) show exact
			changes to tasks/lessons.md (4) request operator confirmation (5) only
			after confirmation produce final updates. Never update files automatically.

		Purpose:
			Capture durable learning and reusable engineering rules.
			Turn errors into durable engineering behavior.

		Responsibility:
			convert mistakes/surprises into reusable rules
			prevent repeated errors
			improve operator discipline (system thinking, safer execution, debugging)
			maintain durable knowledge: concise, actionable, reusable, cross-task

		Involvement at two points:
			Early (State 2 — Context loaded): Prevention mode. Read prior lessons;
			inject relevant warnings into the upcoming task; highlight known pitfalls;
			influence task execution behavior. (Lessons Agent is Secondary in State 2.)
			Late (State 7 — Learning captured): Learning mode. Analyze what happened;
			identify mistakes or surprises; extract reusable rules; propose updates
			to tasks/lessons.md (then wait for operator confirmation before writing).

		Reads:
			tasks/task_XXXX.md (or tasks/todo.md task spec)
			docs/build-reports/TASK-XXXX.md
			docs/review-reports/TASK-XXXX.md  (if present)
			docs/engineering-log.md
			tasks/lessons.md
			verification notes and any corrections made during review
			If repo access is unavailable, request these from the operator.

		Writes/Updates (only after operator confirmation):
			tasks/lessons.md

		Lesson categories (use when adding lessons):
			Operational (command usage, execution mistakes, verification gaps)
			Architectural (design decisions, system boundaries, incorrect assumptions)
			Infrastructure (Linux, storage, networking, Kubernetes)
			Process / Workflow (agent coordination, missing steps, sequencing)
			Safety / Risk (data loss, security gaps, unsafe defaults)

		Lesson format per entry:
			[CATEGORY][ID] Title
			Context / Issue / Root Cause / Rule / Verification / Tags

		Early injection format (State 2): output "Relevant Lessons for Upcoming Task"
			(list lessons + rules), "Key Warnings", "Recommended Precautions".

		Late capture format (State 7): output "Lessons Proposal" with task id, new
			lessons identified, exact proposed entries for tasks/lessons.md, and
			"Operator Confirmation Required: Proceed with updating lessons.md? (Yes/No)".

		What makes a good lesson: reusable, concise, actionable, based on real events,
			prevents future mistakes. Avoid vague statements, one-off observations,
			restating what already exists.

		Must not: implement features; modify architecture; replace Reviewer or Planner;
			update files without confirmation; record unverified assumptions.



4. The workflow: one task moves through fixed states - This is the heart of it.
		(Every meaningful task should go through this sequence)


	State 0 - Strategic Positioning

		Purpose:
			Clarify where the project is overall before discussing any task

		Agent(s):
			Planner Agent (PRIMARY)
			Context Curator (Secondary)

		Reads: 

			tasks/roadmap.md
			context/project-context.json
			context/current-focus.md

		Writes:
			tasks/roadmap.md if phase priorities shift
			context/current-focus.md


		Output:
			current phase
			active milestone
			what kinds of tasks are allowed now
			what is explicitly deferred


		This prevents “developer reflex” from jumping ahead.



	State 1 - Opportunity Captured

		Purpose:
			Record an idea/improvement without forcing immediate execution.

		Agent(s):
			Planner Agent (PRIMARY)
	
		Reads:
			tasks/backlog.md
			tasks/roadmap.md

		Writes:
			tasks/backlog.md


		Output:
			A backlog item with:
				title
				rationale
				category
				possible phase
				not-yet-scoped note

		This is where new ideas go first 
		unless they are truly urgent and phase-aligned.



	State 2 — Context loaded 
		
		Purpose: 
			Understand the task in the current reality of the project before 				planning or building.

		Agent(s): 
			Context Curator Agent (PRIMARY)
			Lessons Agent (Secondary - if relevant lessons exist)

		Reads:
			context/project-context.json
			context/current-focus.md
			context/roadmap.md
			docs/system-blueprint.md
			relevant ADRs			
			tasks/lessons.md

		Writes - Usually nothing mandatory, but may update:

			context/current-focus.md 	(if task priority/focus clarified)
			context/roadmap.md	 	(if significant change is to be updated)
			a short 'context summary' attached to the task

		Output:
			current system state relevant to the task
			assumptions
			constraints
			relevant lessons (if existing) 
			impacted architecture areas (relevant ADRs)

		task is understood in current system reality




	State 3 — Task Specification scoped


		Purpose:
			Promote something from roadmap/backlog into ready execution.
			Understand the task in the current reality of the project before 				planning or building.

		Agent(s):
			Planner Agent (PRIMARY)
			Context Curator Agent (Secondary)

		Reads:
			tasks/roadmap.md
			tasks/backlog.md
			context/current-focus.md

			State 2 context output
			relevant blueprint/docs

		Writes/Updates:
			tasks/todo.md
			possibly mark or annotate item in tasks/backlog.md
			docs/adr/... (optionally if decision-heavy)
			
		Output:
			A scoped task spec containing:
				task id
				goal
				why now
				phase 
				scope / non-scope
				dependencies 
				files & docs affected/expected to change
				risks
				acceptance criteria
				verification steps

			(This goes into tasks/todo.md)


	State 4 — Implementation		

		Purpose:
			Execute the task within the approved scope 

		Agent(s):
			Builder Agent (PRIMARY)
			None during execution, unless implementation is split into subparts

		Reads:
			tasks/todo.md
			context/project-context.json
			docs/system-blueprint.md
			tasks/lessons.md
			other relevant technical/architecture docs (docs/)

		Writes:
			code
			manifests
			scripts
			configs
			docs/build-reports/TASK-xxxx.md
	
		Output:
			implementation
			build report 
			explicit summary of changes and assumptions	
			assumptions made during coding
			proposed files that now require doc updates


		When one task needs more than one Builder:
			For bigger tasks, State 3 can be split into sub-builders:
			Infra Builder for k3s/manifests/secrets/networking
			App Builder for AlethosAPI service/workers
			Data Builder for schema/ingestion/normalization
			Observability Builder for Prometheus/Grafana/alerts


		But they should still all work from the same task spec and converge back into one verification state (State 4)



	State 5 — Verification

		Purpose: 
			Prove the tasks works and does not violate platform principles
			
			Checks:
				does this solve the task?
				does it break architecture?
				does it violate lessons?
				what verification proves it works?
			
		Agent(s):
			Reviewer-Verifier Agent (PRIMARY)
			Builder Agent (Secondary - only to answer clarification/fix issues)


		Reads:
			original task specification
			implementation diff
			build-report
			blueprint
			lessons
			log/tests/output

		Writes: 
			verification report
			review notes
			required fixes
			optionally updates task status in tasks/todo.md
		

		Output:
			pass / revise / fail
			risks found
			missing checks
			security/ops concerns
			architecture fit assessment
			documentation updates required


		It asks: "Would a staff engineer approve this?"




	State 6 — Project State Sync
	
		Purpose: 
			Update repo truth so documentation matches reality.
			Bring docs and context in line with the accepted implementation

		Agent(s):
			Documentation Agent (PRIMARY)
			Context Curator (Secondary)

		Reads:
			accepted implementation
			verification report
			task specification
			changed files
			architecture impact

		Writes:		
			docs/system-blueprint.md
			docs/platform-overview.md 		(if affected)
			docs/security/... 			(if affected)
			context/project-context.json 		(if system state changed)
			ADRs if a meaningful decision was made
			lessons updated 			(if something new was learned)		
			docs/engineering-log/...

	
		Output:
			repo truth updated
			architecture state preserved for future sessions and tools
			explicit record of architectural changes
			synced project context


		This is the most important anti-drift state. 

	


	State 7 — Learning Captured

		Purpose:
			Extract durable learning from task

		Agent(s):
			Lessons Agent (PRIMARY)
			Reviewer-Verifier Agent (Secondary)

		
		Reads:
			verification report
			task outcome (build-report)
			updated docs/context
			engineering log
			any corrections made during review


		Writes:		
			tasks/lessons.md


		Output:
			new lesson entry
			reusable principle
			warning/pattern for future work
						

	State 8 — Task closed / Roadmap Reconciled

		Purpose:
			Close the task and update execution planning 
			Finalize the task only after implementation, verification, and documentation are all complete 

		Agent(s):
			Planner Agent (PRIMARY)
			Context Curator (Secondary)

		
		Reads:
			completed task
			updated docs/context
			roadmap		
			backlog
			lessons


		Writes/Updates:		
			tasks/todo.md 	status = complete
			context/current-focus.md
			tasks/roadmap.md (optionally)
			optionally move follow-up ideas into tasks/backlog.md
	
		Output:
			task marked officially complete
			next focus clarified
			roadmap/backlog adjusted if the completed work changed priorities
				

		Only after:
			implementation exists
			verification is recorded
			docs are updated
			state files reflect reality


	That is the full loop.



Strong operational rule:
	A task is not done when the Builder finishes.

	A task is done only when:
		Reviewer passes it,
		Documentation Agent updates the repo truth,
		Planner closes it,
		Lessons Agent records the learning if needed.





Condensed state-to-agent map

| State | Purpose                          | Primary Agent       | Secondary Agent | Main Files                                                                |
| ----- | -------------------------------- | ------------------- | --------------- | ------------------------------------------------------------------------- |
| 0     | Strategic positioning            | Planner             | Context Curator | `roadmap.md`, `current-focus.md`, `project-context.json`                  |
| 1     | Opportunity captured             | Planner             | —               | `backlog.md`                                                              |
| 2     | Context loaded                   | Context Curator     | Lessons Agent   | `project-context.json`, `system-blueprint.md`, `lessons.md`, `roadmap.md` |
| 3     | Task selected/scoped             | Planner             | Context Curator | `todo.md`, `backlog.md`, `roadmap.md`                                     |
| 4     | Implementation                   | Builder             | —               | code/config + `build-reports/`                                            |
| 5     | Verification                     | Reviewer            | Builder         | task spec, build report, blueprint                                        |
| 6     | Project state sync               | Documentation Agent | Context Curator | docs, context, ADRs                                                       |
| 7     | Learning captured                | Lessons Agent       | Reviewer        | `lessons.md`                                                              |
| 8     | Task closed / roadmap reconciled | Planner             |                 |                                                                           |





5. Canonical Agent Contract

	Input package for any agent
		When invoking an agent, it should always receive:

			current phase
			task id / task spec
			relevant section of project-context.json
			relevant section of system-blueprint.md
			relevant lessons from tasks/lessons.md
			constraints
			expected output format

		It should prevent drift at all costs


	Output package from any agent
		Every non-trivial output should include:
			summary of what changed
			assumptions made
			files that should be updated
			risks / open questions
			whether system-blueprint.md must change
			whether lessons.md should be updated

		State awareness at every step.





6. Foreseen practical workflow 


	Mode A — Planning session
		ChatGPT:
			read context
			refine roadmap
			produce or update task specs
			identify docs to update

	Mode B — Build session
		ChatGPT or Cursor:
			implement one scoped task
			generate code/config
			explain each part

	Mode C — Review session
		ChatGPT as reviewer:
			verify architecture fit
			identify security and ops risks
			determine what docs/lessons change

	Mode D — Documentation session
		ChatGPT to update:
			blueprint
			logs
			lessons
			ADRs


One repo, one state model, many role-based passes:


	speed from AI
	context continuity
	architectural control
	documentation by design
	learning with structured awareness

The repo is the coordination plane.



