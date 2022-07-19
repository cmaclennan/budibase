<script>
  import {
    Layout,
    Heading,
    Body,
    Button,
    List,
    ListItem,
    Modal,
    notifications,
    Pagination,
    Icon,
  } from "@budibase/bbui"
  import { onMount } from "svelte"

  import RoleSelect from "components/common/RoleSelect.svelte"
  import { users, groups, apps, auth } from "stores/portal"
  import AssignmentModal from "./AssignmentModal.svelte"
  import { createPaginationStore } from "helpers/pagination"
  import { Constants } from "@budibase/frontend-core"
  import { roles } from "stores/backend"

  export let app
  let assignmentModal
  let appGroups = []
  let appUsers = []
  let prevSearch = undefined,
    search = undefined
  let pageInfo = createPaginationStore()

  $: page = $pageInfo.page
  $: fetchUsers(page, search)

  $: isProPlan = $auth.user?.license.plan.type !== Constants.PlanType.FREE

  $: appUsers =
    $users.data?.filter(x => {
      return Object.keys(x.roles).find(y => {
        return y === app.prodId
      })
    }) || []

  $: appGroups = $groups.filter(x => {
    return x.apps.find(y => {
      return y.appId === app.appId
    })
  })

  async function addData(appData) {
    let gr_prefix = "gr"
    let us_prefix = "us"
    appData.forEach(async data => {
      if (data.id.startsWith(gr_prefix)) {
        let matchedGroup = $groups.find(group => {
          return group._id === data.id
        })
        matchedGroup.apps.push(app)
        matchedGroup.roles[app.prodId] = data.role

        groups.actions.save(matchedGroup)
      } else if (data.id.startsWith(us_prefix)) {
        let matchedUser = $users.data.find(user => {
          return user._id === data.id
        })

        let newUser = {
          ...matchedUser,
          roles: { [app.prodId]: data.role, ...matchedUser.roles },
        }

        await users.save(newUser)
      }
    })
    await groups.actions.init()
    await users.search({ page, appId: app.prodId })
  }

  async function removeUser(user) {
    // Remove the user role
    const filteredRoles = { ...user.roles }
    delete filteredRoles[app?.prodId]
    await users.save({
      ...user,
      roles: {
        ...filteredRoles,
      },
    })
    await users.search({ page, appId: app.prodId })
  }

  async function removeGroup(group) {
    // Remove the user role

    let filteredApps = group.apps.filter(x => x.appId !== app.appId)
    const filteredRoles = { ...group.roles }
    delete filteredRoles[app?.prodId]

    await groups.actions.save({
      ...group,
      apps: filteredApps,
      roles: { ...filteredRoles },
    })

    await users.search({ page, appId: app.prodId })
  }

  async function updateUserRole(role, user) {
    user.roles[app.prodId] = role
    users.save(user)
  }

  async function updateGroupRole(role, group) {
    group.roles[app.prodId] = role
    groups.actions.save(group)
  }

  async function fetchUsers(page, search) {
    if ($pageInfo.loading) {
      return
    }
    // need to remove the page if they've started searching
    if (search && !prevSearch) {
      pageInfo.reset()
      page = undefined
    }
    prevSearch = search
    try {
      pageInfo.loading()
      await users.search({ page, appId: app.prodId })
      pageInfo.fetched($users.hasNextPage, $users.nextPage)
    } catch (error) {
      notifications.error("Error getting user list")
    }
  }

  onMount(async () => {
    try {
      await groups.actions.init()
      await apps.load()
      await roles.fetch()
    } catch (error) {
      notifications.error("Error")
    }
  })
</script>

<div class="access-tab">
  <Layout>
    {#if appGroups.length || appUsers.length}
      <div>
        <Heading>Access</Heading>
        <div class="subtitle">
          <Body size="S">
            Assign users to your app and define their access here</Body
          >
          <Button on:click={assignmentModal.show} icon="User" cta
            >Assign users</Button
          >
        </div>
      </div>
      {#if isProPlan && appGroups.length}
        <List title="User Groups">
          {#each appGroups as group}
            <ListItem
              title={group.name}
              icon={group.icon}
              iconBackground={group.color}
            >
              <RoleSelect
                on:change={e => updateGroupRole(e.detail, group)}
                autoWidth
                quiet
                value={group.roles[
                  Object.keys(group.roles).find(x => x === app.prodId)
                ]}
              />
              <Icon
                on:click={() => removeGroup(group)}
                hoverable
                size="S"
                name="Close"
              />
            </ListItem>
          {/each}
        </List>
      {/if}
      {#if appUsers.length}
        <List title="Users">
          {#each appUsers as user}
            <ListItem title={user.email} avatar>
              <RoleSelect
                on:change={e => updateUserRole(e.detail, user)}
                autoWidth
                quiet
                value={user.roles[
                  Object.keys(user.roles).find(x => x === app.prodId)
                ]}
              />
              <Icon
                on:click={() => removeUser(user)}
                hoverable
                size="S"
                name="Close"
              />
            </ListItem>
          {/each}
        </List>
        <div class="pagination">
          <Pagination
            page={$pageInfo.pageNumber}
            hasPrevPage={$pageInfo.loading ? false : $pageInfo.hasPrevPage}
            hasNextPage={$pageInfo.loading ? false : $pageInfo.hasNextPage}
            goToPrevPage={pageInfo.prevPage}
            goToNextPage={pageInfo.nextPage}
          />
        </div>
      {/if}
    {:else}
      <div class="align">
        <Layout gap="S">
          <Heading>No users assigned</Heading>
          <div class="opacity">
            <Body size="S"
              >Assign users to your app and set their access here</Body
            >
          </div>
          <div class="padding">
            <Button on:click={() => assignmentModal.show()} cta icon="UserArrow"
              >Assign Users</Button
            >
          </div>
        </Layout>
      </div>
    {/if}
  </Layout>
</div>

<Modal bind:this={assignmentModal}>
  <AssignmentModal {app} {appUsers} {addData} />
</Modal>

<style>
  .access-tab {
    max-width: 600px;
    margin: 0 auto;
    padding: 40px;
  }

  .padding {
    margin-top: var(--spacing-m);
  }
  .opacity {
    opacity: 0.8;
  }

  .align {
    text-align: center;
  }
  .subtitle {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
  }
</style>