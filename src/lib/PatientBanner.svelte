<script lang="ts">
    import axios from 'axios';
    import type { Patient } from "fhir/r4";
    import { onMount } from "svelte";

    export let accessToken: string;
    export let baseURL:string
    export let patient: string
    let patientRersource: Patient | undefined = undefined
    // You can now use the patient variable in your component


    onMount(async () => {
        if (accessToken && baseURL) {
        try {
            const response = await axios.get(`${baseURL}/Patient/${patient}`, {
                headers: {
                    'Authorization': `Bearer ${accessToken}`
                }
            });
            patientRersource = response.data;
            console.log('Patient resource fetched:', patientRersource);
        } catch (error) {
            console.error('Error fetching patient resource:', error);
        }
        }
    });


    
</script>

<style>
  

  
</style>

{#if patientRersource}
  <div class="bg-blue-500 text-white p-4 text-center fixed top-0 left-0 w-full z-50">
    <p class="text-lg">Patient Name: {patientRersource?.name?.[0]?.text}</p>
    <p class="text-lg">Patient Birth Date: {patientRersource.birthDate}</p>
    <p class="text-lg">Patient Gender: {patientRersource.gender}</p>
    
  </div>

 
{/if}

