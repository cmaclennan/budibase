<script>
  import {
    ActionButton,
    Button,
    DatePicker,
    Divider,
    Layout,
    Modal,
    notifications,
    Pagination,
    Select,
    Heading,
    Body,
    Tags,
    Tag,
    Table,
    Page,
  } from "@budibase/bbui"
  import { backups, licensing, auth, admin } from "stores/portal"
  import { createPaginationStore } from "helpers/pagination"
  import AppSizeRenderer from "./AppSizeRenderer.svelte"
  import CreateBackupModal from "./CreateBackupModal.svelte"
  import ActionsRenderer from "./ActionsRenderer.svelte"
  import DateRenderer from "./DateRenderer.svelte"
  import UserRenderer from "./UserRenderer.svelte"
  import StatusRenderer from "./StatusRenderer.svelte"
  import TypeRenderer from "./TypeRenderer.svelte"
  import BackupsDefault from "assets/backups-default.png"
  import { onMount } from "svelte"

  export let app

  let backupData = null
  let modal
  let pageInfo = createPaginationStore()
  let filterOpt = null
  let startDate = null
  let endDate = null
  let filters = getFilters()

  $: page = $pageInfo.page
  $: fetchBackups(filterOpt, page, startDate, endDate)

  function getFilters() {
    const options = []
    let types = ["backup"]
    let triggers = ["manual", "publish", "scheduled", "restoring"]
    for (let type of types) {
      for (let trigger of triggers) {
        let label = `${trigger} ${type}`
        label = label.charAt(0).toUpperCase() + label?.slice(1)
        options.push({ label, value: { type, trigger } })
      }
    }
    options.push({
      label: `Manual restore`,
      value: { type: "restore", trigger: "manual" },
    })
    return options
  }

  const schema = {
    type: {
      displayName: "Type",
    },
    createdAt: {
      displayName: "Date",
    },
    name: {
      displayName: "Name",
    },
    appSize: {
      displayName: "App size",
    },
    createdBy: {
      displayName: "User",
    },
    status: {
      displayName: "Status",
    },
    actions: {
      displayName: null,
    },
  }

  const customRenderers = [
    { column: "appSize", component: AppSizeRenderer },
    { column: "actions", component: ActionsRenderer },
    { column: "createdAt", component: DateRenderer },
    { column: "createdBy", component: UserRenderer },
    { column: "status", component: StatusRenderer },
    { column: "type", component: TypeRenderer },
  ]

  function flattenBackups(backups) {
    return backups.map(backup => {
      return {
        ...backup,
        ...backup?.contents,
      }
    })
  }

  async function fetchBackups(filters, page, startDate, endDate) {
    const response = await backups.searchBackups({
      appId: app.instance._id,
      ...filters,
      page,
      startDate,
      endDate,
    })
    pageInfo.fetched(response.hasNextPage, response.nextPage)

    // flatten so we have an easier structure to use for the table schema
    backupData = flattenBackups(response.data)
  }

  async function createManualBackup(name) {
    try {
      let response = await backups.createManualBackup({
        appId: app.instance._id,
        name,
      })
      await fetchBackups(filterOpt, page)
      notifications.success(response.message)
    } catch {
      notifications.error("Unable to create backup")
    }
  }

  async function handleButtonClick({ detail }) {
    if (detail.type === "backupDelete") {
      await backups.deleteBackup({
        appId: app.instance._id,
        backupId: detail.backupId,
      })
      await fetchBackups(filterOpt, page)
    } else if (detail.type === "backupRestore") {
      await backups.restoreBackup({
        appId: app.instance._id,
        backupId: detail.backupId,
        name: detail.restoreBackupName,
      })
      await fetchBackups(filterOpt, page)
    } else if (detail.type === "backupUpdate") {
      await backups.updateBackup({
        appId: app.instance._id,
        backupId: detail.backupId,
        name: detail.name,
      })
      await fetchBackups(filterOpt, page)
    }
  }

  onMount(() => {
    fetchBackups(filterOpt, page, startDate, endDate)
  })
</script>

<div class="root">
  {#if !$licensing.backupsEnabled}
    <Page wide={false}>
      <Layout gap="XS" noPadding>
        <div class="title">
          <Heading size="M">Backups</Heading>
          <Tags>
            <Tag icon="LockClosed">Pro plan</Tag>
          </Tags>
        </div>
        <div>
          <Body>
            Backup your apps and restore them to their previous state.
            {#if !$auth.accountPortalAccess && !$licensing.groupsEnabled && $admin.cloud}
              Contact your account holder to upgrade your plan.
            {/if}
          </Body>
        </div>
        <Divider />
        <div class="pro-buttons">
          {#if $auth.accountPortalAccess}
            <Button
              newStyles
              primary
              disabled={!$auth.accountPortalAccess && $admin.cloud}
              on:click={$licensing.goToUpgradePage()}
            >
              Upgrade
            </Button>
          {/if}
          <!--Show the view plans button-->
          <Button
            newStyles
            secondary
            on:click={() => {
              window.open("https://budibase.com/pricing/", "_blank")
            }}
          >
            View Plans
          </Button>
        </div>
      </Layout>
    </Page>
  {:else if backupData?.length > 0}
    <Layout noPadding gap="M" alignContent="start">
      <div class="search">
        <div class="select">
          <Select
            placeholder="All"
            label="Type"
            options={filters}
            getOptionValue={filter => filter.value}
            getOptionLabel={filter => filter.label}
            bind:value={filterOpt}
          />
        </div>
        <div>
          <DatePicker
            range={true}
            label={"Filter Range"}
            on:change={e => {
              if (e.detail[0].length > 1) {
                startDate = e.detail[0][0].toISOString()
                endDate = e.detail[0][1].toISOString()
              }
            }}
          />
        </div>

        <div class="split-buttons">
          <ActionButton on:click={modal.show} icon="SaveAsFloppy"
            >Create new backup</ActionButton
          >
        </div>
      </div>
      <div>
        <Table
          {schema}
          allowSelectRows={false}
          allowEditColumns={false}
          allowEditRows={false}
          data={backupData}
          {customRenderers}
          placeholderText="No backups found"
          border={false}
          on:buttonclick={handleButtonClick}
        />
        <div class="pagination">
          <Pagination
            page={$pageInfo.pageNumber}
            hasPrevPage={$pageInfo.loading ? false : $pageInfo.hasPrevPage}
            hasNextPage={$pageInfo.loading ? false : $pageInfo.hasNextPage}
            goToPrevPage={pageInfo.prevPage}
            goToNextPage={pageInfo.nextPage}
          />
        </div>
      </div>
    </Layout>
  {:else if backupData?.length === 0}
    <Page wide={false}>
      <div class="align">
        <img
          width="200px"
          height="120px"
          src={BackupsDefault}
          alt="BackupsDefault"
        />
        <Layout gap="S">
          <Heading>You have no backups yet</Heading>
          <div class="opacity">
            <Body size="S">You can manually backup your app any time</Body>
          </div>
          <div class="padding">
            <Button on:click={modal.show} cta>Create Backup</Button>
          </div>
        </Layout>
      </div>
    </Page>
  {/if}
</div>

<Modal bind:this={modal}>
  <CreateBackupModal {createManualBackup} />
</Modal>

<style>
  .root {
    display: grid;
    grid-template-columns: 1fr;
    height: 100%;
    padding: var(--spectrum-alias-grid-gutter-medium)
      var(--spectrum-alias-grid-gutter-large);
  }

  .search {
    display: flex;
    gap: var(--spacing-xl);
    width: 100%;
    align-items: flex-end;
  }

  .select {
    flex-basis: 150px;
  }

  .pagination {
    display: flex;
    flex-direction: row;
    justify-content: flex-end;
    margin-top: var(--spacing-xl);
  }

  .split-buttons {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    flex: 1;
    gap: var(--spacing-xl);
  }

  .title {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: var(--spacing-m);
  }

  .align {
    margin-top: 5%;
    text-align: center;
  }

  .pro-buttons {
    display: flex;
    gap: var(--spacing-m);
  }
</style>