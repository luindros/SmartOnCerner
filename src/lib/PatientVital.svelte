<script lang="ts">
    import { onMount } from 'svelte';
    import axios from 'axios';
    import type { Bundle, Observation, OperationOutcome, BundleEntry } from "fhir/r4";

    export let accessToken: string;
    export let baseURL: string;
    export let patient: string;

    let observationsArray: Observation[] = []; // Array to hold extracted observations
    let loading = true; // Loading state
    let error: string | null = null; // Error state
    let temperature: number = 0; // State variable for temperature input
    let observationPosting: boolean = false; // State to track if observation is being posted

    // Function to fetch observations from the FHIR API
    const fetchObservations = async (): Promise<Bundle<Observation | OperationOutcome> | null> => {
        try {
            const response = await axios.get(`${baseURL}/Observation`, {
                params: {
                    patient: patient,
                    sort: '-date',
                    category: 'vital-signs'
                },
                headers: {
                    'Authorization': `Bearer ${accessToken}`
                }
            });
            console.log('responseResp', response.data)
            return response.data as Bundle<Observation | OperationOutcome>; // Ensure correct type
        } catch (err) {
            console.error("Error fetching observations:", err);
            return null; // Return null on error
        }
    };

    // Type guard to check if an entry is an Observation
    const isObservation = (entry: BundleEntry<Observation | OperationOutcome>): entry is BundleEntry<Observation> => {
        return entry.resource?.resourceType === 'Observation'; // Check if resource type is 'Observation'
    };

    // Load observations on component mount
    const loadObservations = async () => {
        loading = true; // Set loading state
        error = null; // Reset error state
        const bundle = await fetchObservations(); // Await the promise to fetch observations

        if (bundle && bundle.entry) {
            // Extract observations from the bundle using the type guard
            observationsArray = bundle.entry
                .filter(isObservation) // Filter for entries that are observations
                .map(entry => entry.resource as Observation); // Extract the resource as Observation
        } else {
            error = "No observations found."; // Set error message if no observations
        }

        loading = false; // Reset loading state
    };

    // Function to handle temperature submission
    const postTemperature = async (temp: number) => {
        observationPosting = true; // Set posting state to true
        // Create a new observation for temperature
        const newObservation = {
            resourceType: "Observation",
            status: "final",
            category: [
                {
                    coding: [
                        {
                            system: "http://terminology.hl7.org/CodeSystem/observation-category",
                            code: "vital-signs",
                            display: "Vital Signs"
                        }
                    ],
                    text: "Vital Signs"
                }
            ],
            code: {
                coding: [
                    {
                        system: "http://loinc.org",
                        code: "8331-1", // LOINC code for oral temperature
                        display: "Temperature Oral"
                    }
                ],
                text: "Temperature Oral"
            },
            subject: {
                reference: `Patient/${patient}` // Ensure patientId is used correctly
            },
            effectiveDateTime: new Date().toISOString(), // Current date and time in ISO format
            valueQuantity: {
                value: temp, // The temperature value
                unit: "degC", // Unit of measurement
                system: "http://unitsofmeasure.org", // Unit system
                code: "Cel" // Code for Celsius
            },
        };

        try {
            // Send the new observation to the FHIR API
            await axios.post(`${baseURL}/Observation`, newObservation, {
                headers: {
                    'Authorization': `Bearer ${accessToken}`,
                    'Content-Type': 'application/json'
                }
            });
            // Reset the temperature input after submission
            temperature = 0; // Reset to a default number value
            // Reload observations to include the new entry
            await loadObservations();
        } catch (err) {
            console.error("Error submitting temperature:", err);
            error = "Failed to submit temperature."; // Set error message on failure
        } finally {
            observationPosting = false; // Reset posting state
        }
    };

    onMount(loadObservations); // Call the async function on component mount
</script>


<main>
    <div class="mt-24 mb-2">
        <p class="font-semibold text-gray-700">
            Create new temperature (degree Celsius)
        </p>
        <form on:submit|preventDefault={() => postTemperature(temperature)}>
            <div>
                <input 
                    step=".1" 
                    min="20" 
                    max="50" 
                    bind:value={temperature} 
                    type="number" 
                    class="border border-black p-2 w-48"
                    placeholder="Enter temperature"
                />
                {#if observationPosting}
                    <p class="p-2">Creating observation...</p>
                {:else}
                    <button type="submit" class="bg-black text-white p-2">Submit</button>
                {/if}
            </div>
        </form>
    </div>

    {#if loading}
        <p>Loading observations...</p> <!-- Display loading message -->
    {:else if error}
        <p class="error">{error}</p> <!-- Display error message if any -->
    {:else if observationsArray.length > 0}
        <div class="overflow-x-auto">
            <table class="min-w-full bg-white border border-gray-300 shadow-md rounded-lg">
                <thead class="bg-gray-200">
                    <tr class="text-gray-700">
                        <th class="py-3 px-4 border-b text-left font-semibold">Resource Type</th>
                        <th class="py-3 px-4 border-b text-left font-semibold">ID</th>
                        <th class="py-3 px-4 border-b text-left font-semibold">Date</th>
                        <th class="py-3 px-4 border-b text-left font-semibold">Type</th>
                        <th class="py-3 px-4 border-b text-left font-semibold">Value</th>
                    </tr>
                </thead>
                <tbody>
                    {#each observationsArray as observation}
                        <tr class="hover:bg-gray-100 transition duration-200">
                            <td class="py-2 px-4 border-b">{observation.resourceType}</td>
                            <td class="py-2 px-4 border-b">{observation.id}</td>
                            <td class="py-2 px-4 border-b">{observation.effectiveDateTime}</td>
                            <td class="py-2 px-4 border-b">{observation.code?.coding && observation.code.coding.length > 0 ? observation.code.coding[0].display : 'N/A'}</td>
                            <td class="py-2 px-4 border-b">
                                {observation.valueQuantity?.value !== undefined ? observation.valueQuantity.value : 'N/A'} 
                                {observation.valueQuantity?.unit ? observation.valueQuantity.unit : ''} 
                            </td>
                        </tr>
                    {/each}
                </tbody>
            </table>
        </div>
    {:else}
        <p>No observations found for this patient.</p> <!-- Message when no observations are available -->
    {/if}
</main>