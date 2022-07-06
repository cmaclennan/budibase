<script>
  import {
    Layout,
    Heading,
    Body,
    Button,
    Modal,
    Tag,
    Tags,
    notifications,
  } from "@budibase/bbui"
  import { groups } from "stores/portal"
  import { onMount } from "svelte"
  import CreateEditGroupModal from "./_components/CreateEditGroupModal.svelte"

  import UserGroupsRow from "./_components/UserGroupsRow.svelte"

  let modal
  let group = {
    name: "",
    icon: "UserGroup",
    color: "var(--spectrum-global-color-blue-600)",
    users: [],
    apps: [],
  }
  let proPlan = true

  async function deleteGroup(group) {
    try {
      groups.actions.delete(group)
    } catch (error) {
      notifications.error(`Failed to delete group`)
    }
  }

  async function saveGroup(group) {
    console.log(group)
    try {
      await groups.actions.save(group)
    } catch (error) {
      notifications.error(`Failed to save group`)
    }
  }

  onMount(async () => {
    try {
      await groups.actions.init()
    } catch (error) {
      notifications.error("Error getting User groups")
    }
  })
</script>

<Layout noPadding>
  <Layout gap="XS" noPadding>
    <div style="display: flex;">
      <Heading size="M">User groups</Heading>
      {#if !proPlan}
        <Tags>
          <div class="tags">
            <div class="tag">
              <Tag icon="LockClosed">Pro plan</Tag>
            </div>
          </div>
        </Tags>
      {/if}
    </div>
    <Body>Easily assign and manage your users access with User Groups</Body>
  </Layout>
  <div class="align-buttons">
    <Button
      icon={proPlan ? "UserGroup" : ""}
      cta={proPlan}
      on:click={() => modal.show()}
      >{proPlan ? "Create user group" : "Upgrade Account"}</Button
    >
    {#if !proPlan}
      <Button
        secondary
        on:click={() => {
          window.open("https://budibase.com/pricing/", "_blank")
        }}>View Plans</Button
      >
    {/if}
  </div>

  {#if proPlan}
    <div class="groupTable">
      {#each $groups as group}
        <div>
          <UserGroupsRow {saveGroup} {deleteGroup} {group} />
        </div>
      {/each}
    </div>
  {/if}
</Layout>

<Modal bind:this={modal}>
  <CreateEditGroupModal bind:group {saveGroup} />
</Modal>

<style>
  .align-buttons {
    display: flex;
    column-gap: var(--spacing-xl);
  }
  .tag {
    margin-top: var(--spacing-xs);
    margin-left: var(--spacing-m);
  }

  .groupTable {
    display: grid;
    grid-template-rows: auto;
    align-items: center;
    border-bottom: 1px solid var(--spectrum-alias-border-color-mid);
    border-left: 1px solid var(--spectrum-alias-border-color-mid);
    background: var(--spectrum-global-color-gray-50);
  }

  .groupTable :global(> div) {
    background: var(--bg-color);

    height: 70px;
    display: grid;
    align-items: center;
    grid-gap: var(--spacing-xl);
    grid-template-columns: 2fr 2fr 2fr auto;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    padding: 0 var(--spacing-s);
    border-top: 1px solid var(--spectrum-alias-border-color-mid);
    border-right: 1px solid var(--spectrum-alias-border-color-mid);
  }
</style>