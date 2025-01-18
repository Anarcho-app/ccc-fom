<script>
  import { onMount } from 'svelte';
  import { initialFormData, initialOtherValues, sectionTitles, ETHNICITY_DATA, DELINEATION_OPTIONS } from './constants/formData';
  import ColorPicker from './components/ColorPicker.svelte';
  import EthnicitySection from './components/EthnicitySection.svelte';
  import { getUserId, saveFormData, getFormData } from './db';

  let formData = { ...initialFormData };
  let otherValues = { ...initialOtherValues };
  let otherChecked = Object.fromEntries(Object.keys(initialFormData).map(key => [key, false]));
  let openSections = Object.fromEntries(Object.keys(initialFormData).map(key => [key, false]));
  let phenotype = '#888';
  let userId = getUserId();
  let saveStatus = '';

  $: isValid = phenotype !== '#888';
  $: debugData = {
    formData,
    otherValues,
    otherChecked,
    phenotype
  };

  onMount(loadSavedData);

  async function loadSavedData() {
    try {
      const savedData = await getFormData(userId);
      if (savedData) {
        formData = savedData.formData || { ...initialFormData };
        otherValues = savedData.otherValues || { ...initialOtherValues };
        otherChecked = savedData.otherChecked || Object.fromEntries(Object.keys(initialFormData).map(key => [key, false]));
        phenotype = savedData.phenotype || '#888';
      }
    } catch (error) {
      console.error('Error loading saved data:', error);
    }
  }

  function handlePhenotypeChange(event) {
    phenotype = event.detail;
  }

  async function handleSubmit(e) {
    e.preventDefault();
    if (!isValid) {
      saveStatus = 'Please select a phenotype/skin color';
      return;
    }
    
    const dataToSave = {
      formData,
      otherValues,
      phenotype,
      otherChecked,
      timestamp: new Date().toISOString()
    };

    try {
      const saved = await saveFormData(userId, dataToSave);
      if (saved) {
        saveStatus = 'Data saved successfully!';
        setTimeout(() => saveStatus = '', 3000);
      } else {
        throw new Error('Failed to save data');
      }
    } catch (error) {
      console.error('Error saving data:', error);
      saveStatus = 'Failed to save data. Please try again.';
      setTimeout(() => saveStatus = '', 3000);
    }
  }

  function handleSectionChange(event) {
    const { key, values } = event.detail;
    formData[key] = values;
  }

  function handleOtherChange(event) {
    const { key, checked } = event.detail;
    otherChecked[key] = checked;
  }

  function handleOtherValueChange(event) {
    const { key, value } = event.detail;
    otherValues[key] = value;
  }

  const sections = [
    ...Object.entries(ETHNICITY_DATA).map(([key, options]) => ({
      key,
      title: sectionTitles[key],
      options,
      isArray: true
    })),
    {
      key: 'other',
      title: sectionTitles.other,
      options: [],
      isArray: false
    },
    {
      key: 'delineation',
      title: sectionTitles.delineation,
      options: DELINEATION_OPTIONS,
      isArray: true
    }
  ];
</script>

<div class="max-w-2xl mx-auto p-4">
  <form on:submit={handleSubmit} style:--phenotype={phenotype}>
    <h2 class="text-2xl font-bold mb-4">Ethnicity</h2>
    
    <div class="mb-6">
      <span>Phenotype/Skin Color:</span>
      <ColorPicker {phenotype} on:change={handlePhenotypeChange} />
    </div>

    <div class="space-y-4">
      {#each sections as { key, title, options, isArray }}
        <EthnicitySection
          {title}
          {options}
          {key}
          {phenotype}
          {isArray}
          selectedValues={formData[key]}
          otherChecked={otherChecked[key]}
          otherValue={otherValues[key]}
          on:change={handleSectionChange}
          on:otherChange={handleOtherChange}
          on:otherValueChange={handleOtherValueChange}
        />
      {/each}
    </div>

    {#if saveStatus}
      <div class="mt-4 p-2 rounded text-center {saveStatus.includes('success') ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'}">
        {saveStatus}
      </div>
    {/if}

    <!-- Debug section -->
    <div class="mt-4 p-2 bg-gray-100 rounded">
      <details>
        <summary class="cursor-pointer text-sm text-gray-600 hover:text-gray-800">Debug Data</summary>
        <pre class="text-xs overflow-auto mt-2">
          {JSON.stringify(debugData, null, 2)}
        </pre>
      </details>
    </div>

    <button
      type="submit"
      class="w-full p-2 mt-4 rounded text-white transition-opacity hover:opacity-90 shadow"
      style:background={phenotype}
      style:text-shadow="1px 1px #000"
      disabled={!isValid}
    >
      Submit
    </button>
  </form>
</div>

<style>
  :global(:root) {
    --phenotype: #888;
  }

  pre {
    max-height: 300px;
    white-space: pre-wrap;
    word-wrap: break-word;
  }

  details summary::-webkit-details-marker {
    color: #666;
  }

  details[open] summary {
    margin-bottom: 8px;
  }
</style>
