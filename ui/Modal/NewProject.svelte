<script lang="typescript">
  import { createEventDispatcher, onDestroy } from "svelte";
  import { push } from "svelte-spa-router";

  import { DEFAULT_BRANCH_FOR_NEW_PROJECTS } from "../src/config";
  import * as notification from "../src/notification";
  import * as error from "../src/error";
  import * as path from "../src/path";
  import * as remote from "../src/remote";
  import * as urn from "../src/urn";
  import {
    clearLocalState,
    create,
    defaultBranch,
    localState,
    nameValidationStore,
    descriptionValidationStore,
    formatNameInput,
    extractName,
    repositoryPathValidationStore,
    RepoType,
  } from "../src/project";
  import { ValidationStatus } from "../src/validation";
  import * as screen from "../src/screen";
  import type { Settings } from "../src/settings";
  import { dismissRemoteHelperHint, settings } from "../src/session";

  import { Button, Emoji, Input } from "../DesignSystem/Primitive";
  import {
    Dropdown,
    RadioOption,
    RemoteHelperHint,
    Tooltip,
  } from "../DesignSystem/Component";
  import { CSSPosition } from "../src/style";

  let currentSelection: RepoType;
  let nameInput: HTMLInputElement;

  const dispatch = createEventDispatcher();
  let startValidations = false;

  $: isNew = currentSelection === RepoType.New;
  $: isExisting = currentSelection === RepoType.Existing;

  let name = "";
  let description = "";
  let newRepositoryPath = "";
  let existingRepositoryPath = "";

  let nameValidation = nameValidationStore();
  let descriptionValidation = descriptionValidationStore();

  let loading = false;

  const setCurrentSelection = (type: RepoType) => {
    currentSelection = type;
    // Reset validations on selection switch
    nameValidation = nameValidationStore();
    descriptionValidation = descriptionValidationStore();
  };

  const createProject = async () => {
    try {
      loading = true;
      screen.lock();

      const response = await create({
        description,
        defaultBranch: $defaultBranch,
        repo: isNew
          ? { type: RepoType.New, name, path: newRepositoryPath }
          : { type: RepoType.Existing, path: existingRepositoryPath },
      });

      push(path.project(response.urn));
      notification.info(
        `Project ${response.metadata.name} successfully created`
      );
    } catch (err) {
      push(path.profileProjects());
      error.show({
        code: error.Code.ProjectCreationFailure,
        message: `Could not create project: ${urn.shorten(err.message)}`,
        source: err,
      });
    } finally {
      dispatch("hide");
      loading = false;
      screen.unlock();
    }
  };

  // We unlock the screen already after the request, this is just a fail-safe
  // to make sure the screen gets unlocked in any case when the component gets
  // destroyed.
  onDestroy(() => {
    clearLocalState();
    screen.unlock();
  });

  $: pathValidation = repositoryPathValidationStore(isNew);

  $: repositoryPath = isNew ? newRepositoryPath : existingRepositoryPath;

  $: if (repositoryPath.length > 0 || (currentSelection && name.length > 0)) {
    startValidations = true;
  }

  $: name = formatNameInput(name);

  $: if (startValidations) {
    nameValidation.validate(name);
    descriptionValidation.validate(description);
    pathValidation.validate(repositoryPath);
  }

  // Use the directory name for existing projects as the project name.
  $: name = extractName(existingRepositoryPath);

  // Reset the project name when switching between new and existing repo.
  $: isExisting && (name = "");

  // The presence check is outside the validations since we don't want to show the validation error message for it.
  $: validName =
    name.length > 0 && $nameValidation.status === ValidationStatus.Success;

  $: validDescription =
    description.length === 0 ||
    (description.length > 0 &&
      $descriptionValidation.status === ValidationStatus.Success);

  $: disableSubmit =
    !validName ||
    !validDescription ||
    $pathValidation.status !== ValidationStatus.Success ||
    loading;

  $: localStateStore = $localState;
  $: localBranches =
    localStateStore.status === remote.Status.Success
      ? localStateStore.data.branches
      : [];

  $: showRemoteHelper =
    $settings && ($settings as Settings).appearance.hints.showRemoteHelper;
</script>

<style>
  .container {
    width: 37.5rem;
    background: var(--color-background);
    border-radius: 0.5rem;
    padding: 3rem 2rem 2rem 2rem;
  }

  .create-project {
    display: flex;
    flex-direction: column;
    justify-content: center;
    text-align: center;
  }

  .radio-selector {
    margin-bottom: 2rem;
  }

  .btn-container {
    display: flex;
    justify-content: flex-end;
    margin-top: 1rem;
  }

  .double-button {
    display: grid;
    grid-template-columns: auto auto;
    grid-column-gap: 1rem;
  }

  .default-branch-row {
    display: flex;
    align-items: center;
    margin-top: 1rem;
  }
</style>

<div class="container" data-cy="page">
  <div class="create-project" data-cy="create-project">
    <Emoji
      emoji={'🌠'}
      size="huge"
      style="align-self: center; margin-bottom: 1rem;" />
    <h2 style="margin-bottom: 3rem;">Start a new project</h2>

    <div class="radio-selector">
      <RadioOption
        title="Create a new repository"
        active={isNew}
        on:click={ev => {
          ev.stopPropagation();
          setCurrentSelection(RepoType.New);
        }}
        dataCy="new-project">
        <div slot="option-body">
          <Input.Directory
            placeholder="Where to create the repository"
            validation={$pathValidation}
            bind:path={newRepositoryPath}
            on:selected={() => nameInput.focus()} />
          <p
            style="margin-top: 1rem; color: var(--color-foreground-level-6);
            text-align: center">
            A new repository will be created on your machine inside a directory
            and named after the project name.
          </p>
        </div>
      </RadioOption>

      <RadioOption
        title="Continue with an existing repository"
        active={isExisting}
        on:click={ev => {
          ev.stopPropagation();
          setCurrentSelection(RepoType.Existing);
        }}
        dataCy="existing-project">
        <div slot="option-body">
          <Input.Directory
            placeholder="Choose an existing repository"
            validation={$pathValidation}
            bind:path={existingRepositoryPath} />
          <div class="default-branch-row">
            <p
              style="margin-right: 1rem; color: var(--color-foreground-level-6)">
              Default branch
            </p>
            {#if localBranches.length > 0}
              <Dropdown
                style="max-width: 22.9rem;"
                options={localBranches.map(branch => ({
                  variant: 'text',
                  value: branch,
                  title: branch,
                }))}
                bind:value={$defaultBranch} />
            {:else}
              <Dropdown
                style="max-width: 22.9rem;"
                placeholder={DEFAULT_BRANCH_FOR_NEW_PROJECTS}
                options={[]}
                disabled />
            {/if}
          </div>
          <p
            style="margin-top: 1rem; color: var(--color-foreground-level-6);
            text-align: left;">
            This will publish the chosen repository to the Radicle network.
          </p>
        </div>
      </RadioOption>
      {#if showRemoteHelper}
        <RemoteHelperHint on:hide={dismissRemoteHelperHint} />
      {/if}
    </div>

    <Tooltip
      value={isExisting ? 'The project name is taken from the chosen repository' : ''}
      position={CSSPosition.Top}>
      <Input.Text
        placeholder="Project name*"
        dataCy="name"
        bind:value={name}
        bind:inputElement={nameInput}
        validation={$nameValidation}
        disabled={isExisting} />
    </Tooltip>

    <Input.Text
      dataCy="description"
      style="margin-top: 1rem; margin-bottom: 1rem;"
      placeholder="Project description"
      validation={$descriptionValidation}
      bind:value={description} />

    <div class="btn-container">
      <div class="double-button">
        <Button
          dataCy="cancel-button"
          variant="transparent"
          on:click={() => dispatch('hide')}>
          Cancel
        </Button>
        <Button
          dataCy="create-project-button"
          disabled={disableSubmit}
          variant="primary"
          on:click={createProject}>
          Create project
        </Button>
      </div>
    </div>
  </div>
</div>
