---
title: Workflow API Client
apisections: Workflows
markdown2extras: tables, code-friendly
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# sdc-wf-client

This library allows Workflow API clients to manage their own workflows and jobs.
It is a simple abstraction of the SDC internal Workflow API.

# Features

* Load workflow files from disk into WFAPI
* Find a workflow
* Create jobs
* Find jobs

The distinctive feature that this library provides is being able to drop in any
workflow *compatible* javascript file that can be programatically loaded into
WFAPI, so application developers can write their own custom workflows and use
WfClient to create them with a single function call.

# Getting Started

All you need to use (and test) WfClient is a running WFAPI instance. A sample
unit test is provided for more information about how to load workflows and how
to create jobs for them.

    # To test, get the source.
    git clone git@github.com:joyent/sdc-wf-client.git
    cd sdc-wf-client/
    make test

	# To use as a module, add the package to your package.json
    ...
        "wf-client": "",
    ...

    npm install


## Requiring WfClient Into Your Application

WfClient needs a configuration object with the following keys:

| Name          | Type          | Description                                                                                                                       | Required |
| ------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------- | -------- |
| url           | String        | WFAPI location                                                                                                                    | Yes      |
| path          | String        | Path to the workflow files directory                                                                                              | Yes      |
| log           | Bunyan Logger | Bunyan logger instance                                                                                                            | Yes      |
| forceReplace  | Boolean       | Replace workflows if they exist. Defaults to false                                                                                | No       |
| forceMd5Check | Boolean       | Replace workflows if the MD5 hashes of the contents for the local and remote versions of the files don't match. Defaults to false | No       |
| workflows     | Array         | List of workflow names to load                                                                                                    | No       |

In order to require and initialize a new WfClient, we would then do something
along these lines:

	// Require the module
	//
	var WfClient = require('wf-client');

	// Initialize logger if needed
	//
	var log = new Logger({
	  name: 'wf-client',
	  level: 'info',
	  serializers: {
	      err: Logger.stdSerializers.err,
	      req: Logger.stdSerializers.req,
	      res: restify.bunyan.serializers.response
	  }
	});

	// Config objet for WfClient
	//
	var config = {
	    'workflows': ['say'],
	    'url': 'http://10.99.99.15',
	    'path': './test',
	    'log': log
	};

	// Initialize WfClient
	//
	var wfapi = new WfClient(config);

With this configuration, we are telling WfClient that we our workflows directory
is located at './test' and we have a workflow called 'say'. This means that if
we wanted to load the 'say' workflows into WFAPI, WfClient will assume that
there is a say.js javascript file located at './test/say.js'.

It is not entirely need to list all workflows that WfClient needs to be aware of
from the config file. There are methods to add new workflows to WFAPI
programatically.

Note the configuration option called 'forceReplace'. It can be used as a
'dev mode' flag where we are working on constant modifications to workflow files
and would like to replace the workflows on WFAPI automatically on each
application restart. This option is useful to let wf-client take care of
duplicate workflows on WFAPI and replace them every time .initWorkflows() is
called.


# WFClient API

## initWorkflows(callback)

Initializes every workflow listed in the workflows instance variable. The
workflow that a client wishes to use should be present in the WfClient
configuration object as the 'workflows' key. What this function does in the
background is to call wf-client.loadWorkflow for each of the workflows
available, so it is more of a convenience method than anything.

| Name     | Type     | Description |
| -------- | -------- | ----------- |
| callback | Function | fn(err)     |


## loadWorkflow(fileName, callback)

Loads a workflow by fileName. WfClient will try to require the file and check if
the workflow already exists in WFAPI. Note that WfClient relies on workflow.name
and workflow.version to construct the name that the workflow object will have in
WFAPI. As an example, here is a fragment of test workflow named say.js:

	...
	var VERSION = '1.0.0';

	var workflow = module.exports = {
	    name: 'say-' + VERSION,
	    version: VERSION,
	...

WfClient forms the workflow name by concatenating name and version, but users
only need to know about its fileName and this is how they should refer to them.
In our example we will want to load 'say' and not 'say-1.0.0' since say.js is
the workflow file name.

| Name     | Type     | Description                    |
| -------- | -------- | ------------------------------ |
| fileName | String   | Filename/alias of the workflow |
| callback | Function | fn(err)                        |


## findWorkflow(name, callback)

Finds a workflow by name. Used internally by loadWorkflow. Note that in this
case the worflow the name is the **name** attribute of the object, which means
that for the *say* workflow its name would be 'say-1.0.0'. This method
is a convenience method since WFAPI doesn't provide a search by name
(and/or version) at the moment.

| Name     | Type     | Description          |
| -------- | -------- | -------------------- |
| name     | String   | Name of the workflow |
| callback | Function | fn(err, workflow)    |


## getWorkflow(uuid, callback)

Finds a workflow by uuid.

| Name     | Type     | Description          |
| -------- | -------- | -------------------- |
| uuid     | UUID     | UUID of the workflow |
| callback | Function | fn(err, workflow)    |


## createJob(fileName, params, [options], callback)

Queues a new job for the given workflow. Params is the object that is going to
be posted as job parameters. Note also that the workflow name is the fileName
that WfClient already associated to a workflow object internally. fileName can
be omitted if the workflow uuid is given as params.workflow. This means that the
function takes the following two forms:

 * createJob(fileName, params, options, cb)
 * createJob(params, headers, cb)

Options is always an optional argument. It is meant to accept additional options
that should affect the the job creation request. For example, if options.headers
are specified then WfClient will add them to the request headers. This is useful
for clients that need to pass specific information across different APIs in order
to track requests by any given identifier. Only options.headers is a supported
option at the moment

| Name            | Type     | Description                        |
| --------------- | -------- | ---------------------------------- |
| fileName        | String   | Filename/alias of the workflow     |
| params          | Object   | Job parameters                     |
| params.target   | String   | Job target. **Required**           |
| options         | Object   | Additional options for the job     |
| options.headers | Object   | Additional request headers to pass |
| callback        | Function | fn(err, job)                       |

**NOTE REGARDING JOB TARGET** Job target is a way for WFAPI to detect that two
jobs represent the same 'request'. One can use target as a way to identify
unique job instances for when a job might contain the same request parameters,
i.e. by suffixing a timestamp to a user generated target string.

The only rule around job targets is that two jobs with the same params/target
can't be running at the same time. For queuing multiple jobs of the same
workflow a UUID or a timestamp should help identifying them by target.


## getJob(uuid, callback)

Finds a job by uuid.

| Name     | Type     | Description     |
| -------- | -------- | --------------- |
| uuid     | UUID     | UUID of the job |
| callback | Function | fn(err, job)    |


## listJobs(query, callback)

Searches jobs that match the given query conditions. Any job can be searched by
its parameters. The general recommendation is to add custom scopes to jobs in
the form of key/values for their parameters. As an example, one would like to
classify jobs by task, in which case a job can have the following value for the
task key in its parameters:

	var parameters = {
		...
		'task': 'cleanup'
		...
	};

Given this, a way to find all jobs for the task type cleanup would be to issuing
the following function call:

	wf.listJobs( { task: 'cleanup' }, function (err, jobs) {});

| Name     | Type     | Description   |
| -------- | -------- | ------------- |
| query    | Object   | Query options |
| callback | Function | fn(err, jobs) |
