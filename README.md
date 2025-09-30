<p align="center">
    <a alt="License"
        href="https://github.com/coalesceio/jira/blob/main/LICENSE">
        <img src="https://img.shields.io/badge/License-Apache%202.0-blue.svg" /></a>
    <a alt="Maintained?">
        <img src="https://img.shields.io/badge/Maintained%3F-yes-green.svg" /></a>
    <a alt="PRs">
        <img src="https://img.shields.io/badge/Contributions-welcome-blueviolet" /></a>
    <a alt="Fivetran Quickstart Compatible"
</p>

# GA4 Coalesce Pipeline ([Docs](https://github.com/coalesceio/Google4Analytics/))


<!--section="jira_transformation_model"-->
The following table provides a detailed list of all models materialized within this package by default. 

This package connects to an exported GA4 dataset and provides useful transformations as well as report-ready dimensional models that can be used to build reports.

Features include:
- Flattened models to access common events and event parameters such as `page_view`, `session_start`, and `purchase`
- Conversion of sharded event tables into a single partitioned table
- Incremental loading of GA4 data into your staging tables 
- Page, session and user dimensional models with conversion counts
- Last non-direct session attribution
- Simple methods for accessing query parameters (like UTM params) or filtering query parameters (like click IDs)
- Support for custom event parameters & user properties
- Mapping from source/medium to default channel grouping

# Models

| model | description |
|-------|-------------|
| stg_ga4__events | Contains cleaned event data that is enhanced with useful event and session keys. |
| stg_ga4__event_* | 1 model per event (ex: page_view, purchase) which flattens event parameters specific to that event |
| stg_ga4__event_items | Contains item data associated with e-commerce events (Purchase, add to cart, etc) |
| stg_ga4__event_to_query_string_params | Mapping between each event and any query parameters & values that were contained in the event's `page_location` field |
| stg_ga4__user_properties | Finds the most recent occurance of specified user_properties for each user |
| stg_ga4__derived_user_properties | Finds the most recent occurance of specific event_params value and assigns them to a client_key. Derived user properties are specified as variables (see documentation below) |
| stg_ga4__derived_session_properties | Finds the most recent occurance of specific event_params or user_properties value and assigns them to a session's session_key. Derived session properties are specified as variables (see documentation below) |
| stg_ga4__session_conversions_daily | Produces daily counts of conversions per session. The list of conversion events to include is configurable (see documentation below) |
| stg_ga4__sessions_traffic_sources | Finds the first source, medium, campaign, content, paid search term (from UTM tracking), and default channel grouping for each session. |
| stg_ga4__sessions_traffic_sources_daily | Same data as stg_ga4__sessions_traffic_sources, but partitioned by day to allow for efficient loading and querying of data. |
| stg_ga4__sessions_traffic_sources_last_non_direct_daily | Finds the last non-direct source attributed to each session within a 30-day lookback window. Assumes each session is contained within a day. |
| dim_ga4__client_keys | Dimension table for user devices as indicated by client_keys. Contains attributes such as first and last page viewed.| 
| dim_ga4__sessions | Dimension table for sessions which contains useful attributes such as geography, device information, and acquisition data. Can be expensive to run on large installs (see `dim_ga4__sessions_daily`) |
| dim_ga4__sessions_daily | Query-optimized session dimension table that is incremental and partitioned on date. Assumes that each partition is contained within a single day |
| fct_ga4__pages | Fact table for pages which aggregates common page metrics by date, stream_id and page_location. |
| fct_ga4__sessions_daily | Fact table for session metrics, partitioned by date. A single session may span multiple rows given that sessions can span multiple days.  |
| fct_ga4__sessions | Fact table that aggregates session metrics across days. This table is not partitioned, so be mindful of performance/cost when querying. |

# How do I use the Coalesce pipeline?

## Step 1: Prerequisites
To use this Coalesce pipeline, you must have the following:

- Schemas that map to Coalesce Storage Locations
- Coalesce Markeplace Packages (already included in Git repo)
  - [Base Node Types](https://docs.coalesce.io/page/package_coalesce_base-node-types) 

## Step 2: Import the coalesceio GA4 repository

1. In Github select create a new repository and select `Import a repository`.
2. The URL for the GA4 source repository is [https://github.com/coalesceio/jira.git](https://github.com/coalesceio/Google4Analytics.git).  This is a Public repository so no credentials are required.
3. Select an owner and give your repository a name.
4. After the import is complete you will see two branches available:
    - **GA4_full_load** - This is an entire pipeline of mostly views loaded by the job `GA4_Pipeline` to fully reload the GA4 pipeline.
5. Branch can be selected when setting up a Workspace which will be described below.

## Step 3: Set up a Project / Workspace in Coalesce

1. After the Git repo has been imported follow the Coalesce [documentation](https://docs.coalesce.io/docs/projects#create-a-new-project) to create a new project.  Initially, choose the option `Skip and Create` in the window for `Setup Version Control`.  We will connect to the Git repository after creating a Workspace.
Once the Project has been created select `Create Workspace`.  Enter a name and meaninful desription based on the Git branch you want to start from, either Dynamic Table or full load based.
2. At this point we are going to set up version control.  Select `Project Settings` and in the [Git Repository](https://docs.coalesce.io/docs/changing-a-git-repository-in-coalesce) section enter the URL of the repository you imported into your Git account as the Git Repository URL.
3. Save the `Project Settings`.
4. If you have enabled security for your Git repo, [Configure Git Account](https://docs.coalesce.io/docs/set-up-your-git-integration#add-through-the-project-dashboard).
5. After configuring the Git repo select `Launch` to launch the Workspace so we can attach it to a Git branch.
6. A Workspace can be attached to a branch by either selecting the `Git` modal or selecting `git branch` from the Workspace warning message `"Finish setting up version control for this workspace and avoid losing any work. Attach this workspace to a git branch"`.
7. After the `Attach Workspace to Branch` opens select the desired branch - **GA4_full_laod** to attach and `Attach` it.
8. Click on the `Git` modal, navigate to the `Branches` tab and select the [Branch Action](https://docs.coalesce.io/docs/git-branches#branch-actions) `Force Checkout` to populate the workspace with the latest Git commit.
9. This will overwrite any uncommitted work in the Workspace, which is what we want, so you will be required to confirm the Force Checkout by typing **FORCE** in the screen.
10. At this point the DAG objects should appear in your Workspace with errors.  Some workspace configuration is required to fix these errors.

## Step 4: Workspace Configuration

In this section you will configure the following Workspace settings:
- **Build Settings** - Configure [Storage Locations](https://docs.coalesce.io/docs/storage-locations-and-storage-mappings)
- **Workspace** - Configure Settings, User Credentials, Storage Mappings and Parameters

### Build Settings
Build Settings changes are required for the `GA4_full_load` versions of the Google Analytics pipeline.

#### Storage Locations
The pipeline equires four Storage Locations be created.
- **TARGET** (Default Location)** - The final output of the pipeline is stored in the `TGT` location.  
- **SOURCE** - Source location where GA4 data extract is stored


#### Environments
Environments must be configured in order to deploy pipeline to higher level environments (QA, UAT, Pre-Prod, Prod, etc.) based on how you are managing your environments.

If implementing the `GA4_full_load` then no parameters are required.

### Workspace Settings

- **Settings** - Configure the Snowflake account that Coalesce will be utilizing
- **User Credentials / OAuth Settings** - Enter the credentials required to connect to Snowflake
- **Storage Mappings** - This can be configured to use one database / schema for all Storage Locations or up to four database / schema mappings, one for each Storage Location, depending on whether or not you want to seperate Source, Staging, Intermediate and Target objects.


## Full Load DAG Specifics
### Pipeline Definition
The pipeline is comprised of 1 sources, 80 views and 2 Dimensions, 5 Facts.  

### Missing Sources
If there are sources not available in your specific case (SPRINT, COMPONENT, VERSION), nodes related to those areas can be deleted from the pipeline.  This will require modification of any downstream objects that rely on said sources, but should be quick to figure out utilizing Coalesce object and column level lineage capabilities.

### Executing Pipeline

After the Workspace has been configured commit any changes into Git.  If the only changes have been Build Settings and Workspace Settings then there may be nothing to commit.

At this point you can create the pipeline.  The easiest way to do this is select `Create All` from the Graph action menu.

# How is this pipeline maintained and can I contribute?
## Pipeline Maintenance
The Coalesce team maintaining this package _only_ maintains the latest version of the package. 

## Contributions
A small team of analytics engineers at Coalesce develops these pipelines. However, the pipelines are made better by community contributions! 

# Are there any resources available?
- If you have questions or want to reach out for help, please refer to the [GitHub Issue](https://github.com/coalesceio/GA4/issues) section to find the right avenue of support for you.
- If you would like to provide feedback to the Coalesce pipeline team or would like to request a new Coalesce pipeline, fill out our [Feedback Form](https:tbd).
- Any other questions about Coalesce then check out our [Help Center](https://help.coalesce.io/)
