.. _divedeep-admin-service:

#############################
FlyteAdmin Service Background
#############################

Entities
========
The `admin service definition <https://github.com/flyteorg/flyteidl/blob/master/protos/flyteidl/service/admin.proto>`__ defines REST operations for the entities
flyteadmin administers.

As a refresher, the primary :ref:`entities <divedeep>` across Flyte map similarly to FlyteAdmin entities.

Static entities
+++++++++++++++

These include:

- Workflows
- Tasks
- Launch Plans

Permitted operations:

- Create
- Get
- List

The above are designated by an `identifier <https://github.com/flyteorg/flyteidl/blob/793b09d190148236f41ad8160b5cec9a3325c16f/protos/flyteidl/core/identifier.proto#L16>`_
which consists of a project, domain, name and version specification. These entities are for the most part immutable. To update one of these specific entities, the updated
version must be reregistered with a unique and new version identifier attribute.

One caveat is that launch plan state can toggle between `ACTIVE or INACTIVE <https://github.com/flyteorg/flyteidl/blob/6b8e34af67cbd06d2f20a6f7462e12c7f0d34f3b/protos/flyteidl/admin/launch_plan.proto#L33>`_.
At most one launch plan version across a shared project, domain and name specification can be active at a time. The state affects scheduled launch plans only.
An inactive launch plan can still be used to launch individual executions. However, only an active launch plan runs on a schedule (if it has a schedule defined).


Static entities metadata (Named Entities)
+++++++++++++++++++++++++++++++++++++++++
A `named entity <https://github.com/flyteorg/flyteidl/blob/c12816cf3f0bde54a67fa77e71c0158a64151ea3/protos/flyteidl/admin/common.proto#L45>`__ includes metadata for one of the above entities
(workflow, task or launch plan) across versions. A named entity includes a resource type (workflow, task or launch plan) and an
`id <https://github.com/flyteorg/flyteidl/blob/c12816cf3f0bde54a67fa77e71c0158a64151ea3/protos/flyteidl/admin/common.proto#L12>`__ which is composed of project, domain and name.
A named entity also includes metadata, which are mutable attributes about the referenced entity.

This metadata includes:

- Description: a human readable description for the Named Entity collection
- State (workflows only): this determines whether the workflow is shown on the overview list of workflows scoped by project and domain

Permitted operations:

- Create
- Update
- Get
- List


Execution entities
++++++++++++++++++

These include:

- (Workflow) executions
- Node executions
- Task executions

Permitted operations:

- Create
- Get
- List

After an execution begins, flyte propeller monitors the execution and sends events which admin uses to update the above executions. 

These `events <https://github.com/flyteorg/flyteidl/blob/793b09d190148236f41ad8160b5cec9a3325c16f/protos/flyteidl/event/event.proto>`_ include

- WorkflowExecutionEvent
- NodeExecutionEvent
- TaskExecutionEvent

and include information about respective phase transitions, phase transition time and optional output data if the event concerns a terminal phase change.

These events are the **only** way to update an execution. No raw Update endpoint exists.

To track the lifecycle of an execution admin stores attributes such as duration, timestamp at which an execution transitioned to running, and end time.

For debug purposes admin also stores Workflow and Node execution events in its database, but does not currently expose them through an API. Because array tasks can yield very many executions,
admin does **not** store TaskExecutionEvents.


Platform entities
+++++++++++++++++
Projects: like named entities, project have mutable metadata such as human-readable names and descriptions, in addition to their unique string ids.

Permitted project operations:

- Register
- List

Matchable resources
A thorough background on :ref:`matchable resources <howto-managing-customizable-resources>` explains their purpose and application logic. As a summary, these are used to override system level defaults
for kubernetes cluster resource management, default execution values, and more all across different levels of specificity.

These entities consist of:

- ProjectDomainAttributes
- WorkflowAttributes

Where ProjectDomainAttributes configure customizable overrides at the project and domain level and WorkflowAttributes configure customizable overrides at the project, domain and workflow level.

Permitted attribute operations:

- Update (implicitly creates if there is no existing override)
- Get
- Delete

Using the Admin Service
=======================

Adding request filters	
++++++++++++++++++++++	

We use `gRPC Gateway <https://github.com/grpc-ecosystem/grpc-gateway>`_ to reverse proxy http requests into gRPC.	
While this allows for a single implementation for both HTTP and gRPC, an important limitation is that fields mapped to the path pattern cannot be	
repeated and must have a primitive (non-message) type. Unfortunately this means that repeated string filters cannot use a proper protobuf message. Instead use	
the internal syntax shown below::	

 func(field,value) or func(field, value)	

For example, multiple filters would be appended to an http request::	

 ?filters=ne(version, TheWorst)+eq(workflow.name, workflow)	

Timestamp fields use the RFC3339Nano spec (ex: "2006-01-02T15:04:05.999999999Z07:00")	

The fully supported set of filter functions are	

- contains	
- gt (greater than)	
- gte (greter than or equal to)	
- lt (less than)	
- lte (less than or equal to)	
- eq (equal)	
- ne (not equal)	
- value_in (for repeated sets of values)	

"value_in" is a special case where multiple values are passed to the filter expression. For example::	

 value_in(phase, 1;2;3)	

Filterable fields vary based on entity types:	

- Task	

  - project	
  - domain	
  - name	
  - version	
  - created_at	
- Workflow	

  - project	
  - domain	
  - name	
  - version	
  - created_at	
- Launch plans	

  - project	
  - domain	
  - name	
  - version	
  - created_at	
  - updated_at	
  - workflows.{any workflow field above} (for example: workflow.domain)	
  - state (you must use the integer enum e.g. 1)	
     - States are defined in `launch_plan.proto <https://github.com/flyteorg/flyteidl/blob/2c17791170ece1cced3e96daa08ea2692efe3d07/protos/flyteidl/admin/launch_plan.proto#L23>`_.	
- Named Entity Metadata

  - state (you must use the integer enum e.g. 1)	
     - States are defined in `common.proto <https://github.com/flyteorg/flyteidl/blob/c12816cf3f0bde54a67fa77e71c0158a64151ea3/protos/flyteidl/admin/common.proto#L25>`_.
- Executions (Workflow executions)	

  - project	
  - domain	
  - name	
  - workflow.{any workflow field above} (for example: workflow.domain)	
  - launch_plan.{any launch plan field above} (for example: launch_plan.name)	
  - phase (you must use the upper-cased string name e.g. RUNNING)	
     - Phases are defined in `execution.proto <https://github.com/flyteorg/flyteidl/blob/223537e15e05bc6925403a8c11c5a09d91008a80/protos/flyteidl/core/execution.proto#L11,L21>`__.
  - execution_created_at	
  - execution_updated_at	
  - duration (in seconds)	
  - mode (you must use the integer enum e.g. 1)	
     - Modes are defined in `execution.proto <https://github.com/flyteorg/flyteidl/blob/182eeb9e1d0f2369e479a981f30cb51c2d7a0672/protos/flyteidl/admin/execution.proto#L96>`__.
  - user (authenticated user or role from flytekit config)

- Node Executions	

  - node_id	
  - execution.{any execution field above} (for example: execution.domain)	
  - phase (you must use the upper-cased string name e.g. QUEUED)	
     - Phases are defined in `execution.proto <https://github.com/flyteorg/flyteidl/blob/223537e15e05bc6925403a8c11c5a09d91008a80/protos/flyteidl/core/execution.proto#L26,L36>`__.	
  - started_at	
  - node_execution_created_at	
  - node_execution_updated_at	
  - duration (in seconds)	
- Task Executions	

  - retry_attempt	
  - task.{any task field above} (for example: task.version)	
  - execution.{any execution field above} (for example: execution.domain)	
  - node_execution.{any node execution field above} (for example: node_execution.phase)	
  - phase (you must use the upper-cased string name e.g. SUCCEEDED)	
     - Phases are defined in `execution.proto <https://github.com/flyteorg/flyteidl/blob/223537e15e05bc6925403a8c11c5a09d91008a80/protos/flyteidl/core/execution.proto#L42,L49>`__.	
  - started_at	
  - task_execution_created_at	
  - task_execution_updated_at	
  - duration (in seconds)	

Putting it all together	
-----------------------	

If you wanted to do query on specific executions that were launched with a specific launch plan for a workflow with specific attributes, you could do something like:	

::	

   gte(duration, 100)+value_in(phase,RUNNING;SUCCEEDED;FAILED)+eq(lauch_plan.project, foo)	
   +eq(launch_plan.domain, bar)+eq(launch_plan.name, baz)	
   +eq(launch_plan.version, 1234)	
   +lte(workflow.created_at,2018-11-29T17:34:05.000000000Z07:00)	
   	
   	

Adding sorting to requests	
++++++++++++++++++++++++++	

Only a subset of fields are supported for sorting list queries. The explicit list is below:	

- ListTasks	

  - project	
  - domain	
  - name	
  - version	
  - created_at	
- ListTaskIds	

  - project	
  - domain	
- ListWorkflows	

  - project	
  - domain	
  - name	
  - version	
  - created_at	
- ListWorkflowIds	

  - project	
  - domain	
- ListLaunchPlans	

  - project	
  - domain	
  - name	
  - version	
  - created_at	
  - updated_at	
  - state (you must use the integer enum e.g. 1)	
     - States are defined in `launch_plan.proto <https://github.com/flyteorg/flyteidl/blob/2c17791170ece1cced3e96daa08ea2692efe3d07/protos/flyteidl/admin/launch_plan.proto#L23>`_.	
- ListWorkflowIds	

  - project	
  - domain	
- ListExecutions	

  - project	
  - domain	
  - name	
  - phase (you must use the upper-cased string name e.g. RUNNING)	
     - Phases are defined in `execution.proto <https://github.com/flyteorg/flyteidl/blob/223537e15e05bc6925403a8c11c5a09d91008a80/protos/flyteidl/core/execution.proto#L11,L21>`__.	
  - execution_created_at	
  - execution_updated_at	
  - duration (in seconds)	
  - mode (you must use the integer enum e.g. 1)	
     - Modes are defined `execution.proto <https://github.com/flyteorg/flyteidl/blob/182eeb9e1d0f2369e479a981f30cb51c2d7a0672/protos/flyteidl/admin/execution.proto#L96>`__.	
- ListNodeExecutions	

  - node_id	
  - retry_attempt	
  - phase (you must use the upper-cased string name e.g. QUEUED)	
     - Phases are defined in `execution.proto <https://github.com/flyteorg/flyteidl/blob/223537e15e05bc6925403a8c11c5a09d91008a80/protos/flyteidl/core/execution.proto#L26,L36>`__.
  - started_at	
  - node_execution_created_at	
  - node_execution_updated_at	
  - duration (in seconds)	
- ListTaskExecutions	

  - retry_attempt	
  - phase (you must use the upper-cased string name e.g. SUCCEEDED)	
     - Phases are defined in `execution.proto <https://github.com/flyteorg/flyteidl/blob/223537e15e05bc6925403a8c11c5a09d91008a80/protos/flyteidl/core/execution.proto#L42,L49>`__.
  - started_at	
  - task_execution_created_at	
  - task_execution_updated_at	
  - duration (in seconds)	

Sorting syntax	
--------------	

Adding sorting to a request requires specifying the ``key``, e.g. the attribute you wish to sort on. Sorting can also optionally specify the direction (one of ``ASCENDING`` or ``DESCENDING``) where ``DESCENDING`` is the default.	

Example sorting http param:	

::	

   sort_by.key=created_at&sort_by.direction=DESCENDING	
   	
Alternatively, since descending is the default, the above could be rewritten as	

::	

   sort_by.key=created_at

