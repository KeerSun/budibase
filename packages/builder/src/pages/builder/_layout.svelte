<script>
  import { isActive, redirect } from "@roxi/routify"
  import { admin, auth } from "stores/portal"
  import { onMount } from "svelte"

  let loaded = false

  $: multiTenancyEnabled = $admin.multiTenancy
  $: hasAdminUser = !!$admin?.checklist?.adminUser
  $: tenantSet = $auth.tenantSet

  onMount(async () => {
    await admin.init()
    await auth.checkAuth()
    loaded = true
  })

  $: {
    const apiReady = $admin.loaded && $auth.loaded
    // if tenant is not set go to it
    if (loaded && apiReady && multiTenancyEnabled && !tenantSet) {
      $redirect("./auth/org")
    }
    // Force creation of an admin user if one doesn't exist
    else if (loaded && apiReady && !hasAdminUser) {
      $redirect("./admin")
    }
  }

  // Redirect to log in at any time if the user isn't authenticated
  $: {
    if (
      loaded &&
      hasAdminUser &&
      !$auth.user &&
      !$isActive("./auth") &&
      !$isActive("./invite")
    ) {
      const returnUrl = encodeURIComponent(window.location.pathname)
      $redirect("./auth?", { returnUrl })
    } else if ($auth?.user?.forceResetPassword) {
      $redirect("./auth/reset")
    }
  }
</script>

{#if loaded}
  <slot />
{/if}
