<script lang="ts">
    import axios from "axios";
    import type { Bundle, Observation } from "fhir/r4";
    import { onMount } from "svelte";

    export let accessToken: string;
    export let patientId: string;
    export let baseUrl: string;

    let vitalObservations: Bundle | undefined = undefined;
    let loading = true;
    let error: string | null = null;
    let showCreateForm = false;
    let creating = false;
    let createError: string | null = null;
    let createSuccess = false;

    // Form fields
    let temperatureValue = "";

    const getVitals = async (): Promise<Bundle> => {
        try {
            // Fetch ALL vital signs without filtering by specific codes
            const response = await axios.get<Bundle>(
                `${baseUrl}/Observation?patient=${patientId}&category=vital-signs&_sort=-date&_count=200`,
                {
                    headers: {
                        Authorization: `Bearer ${accessToken}`,
                    },
                },
            );
            return response.data;
        } catch (err) {
            throw new Error(`Failed to fetch vital signs: ${err}`);
        }
    };

    const getVitalsEntries = (bundle: Bundle) => {
        return bundle.entry || [];
    };

    const formatDateTime = (dateTime: string | undefined): string => {
        if (!dateTime) return "Unknown date";
        return new Date(dateTime).toLocaleString();
    };

    const bpDisplay = (observation: Observation): string => {
        if (observation.component && observation.component.length > 0) {
            // Handle blood pressure with components
            const systolic = observation.component.find((c) =>
                c.code?.coding?.some((coding) => coding.code === "8480-6"),
            );
            const diastolic = observation.component.find((c) =>
                c.code?.coding?.some((coding) => coding.code === "8462-4"),
            );

            if (systolic?.valueQuantity && diastolic?.valueQuantity) {
                return `${systolic.valueQuantity.value}/${diastolic.valueQuantity.value} ${systolic.valueQuantity.unit || "mmHg"}`;
            }
        }

        // Handle single value observations
        if (observation.valueQuantity) {
            return `${observation.valueQuantity.value} ${observation.valueQuantity.unit || ""}`;
        }

        if (observation.valueString) {
            return observation.valueString;
        }

        return "Not available";
    };

    const createTemperatureObservation = async () => {
        if (!temperatureValue || isNaN(parseFloat(temperatureValue))) {
            createError = "Please enter a valid temperature value";
            return;
        }

        creating = true;
        createError = null;
        createSuccess = false;

        try {
            const temperatureObservation: Observation = {
                resourceType: "Observation",
                status: "final",
                category: [
                    {
                        coding: [
                            {
                                system: "http://terminology.hl7.org/CodeSystem/observation-category",
                                code: "vital-signs",
                                display: "Vital Signs",
                            },
                        ],
                        text: "Vital Signs",
                    },
                ],
                code: {
                    coding: [
                        {
                            system: "http://loinc.org",
                            code: "8331-1",
                        },
                    ],
                    text: "Temperature Oral",
                },
                subject: {
                    reference: `Patient/${patientId}`,
                },
                effectiveDateTime: new Date().toISOString(),
                valueQuantity: {
                    value: parseFloat(temperatureValue),
                    unit: "degC",
                    system: "http://unitsofmeasure.org",
                    code: "Cel",
                },
            };

            console.log(
                "Creating temperature observation:",
                JSON.stringify(temperatureObservation, null, 2),
            );
            console.log("POST URL:", `${baseUrl}/Observation`);
            console.log("Headers:", {
                Authorization: `Bearer ${accessToken}`,
                "Content-Type": "application/fhir+json",
            });

            const response = await axios.post(
                `${baseUrl}/Observation`,
                temperatureObservation,
                {
                    headers: {
                        Authorization: `Bearer ${accessToken}`,
                        "Content-Type": "application/fhir+json",
                    },
                },
            );

            console.log("POST response status:", response.status);
            console.log("POST response headers:", response.headers);
            console.log(
                "POST response data:",
                JSON.stringify(response.data, null, 2),
            );
            console.log("Created observation ID:", response.data?.id);
            console.log("Location header:", response.headers?.location);

            if (response.data) {
                console.log("Full created observation:", response.data);
            } else {
                console.log(
                    "No response body - check Location header for created resource URL",
                );
            }

            createSuccess = true;
            showCreateForm = false;

            // Reset form
            temperatureValue = "";

            // Refresh the vitals list
            vitalObservations = await getVitals();
        } catch (err) {
            console.error("Error creating temperature observation:", err);
            if (axios.isAxiosError(err)) {
                console.error("Response status:", err.response?.status);
                console.error("Response data:", err.response?.data);
                createError = `HTTP ${err.response?.status}: ${err.response?.data?.issue?.[0]?.diagnostics || err.message}`;
            } else {
                createError =
                    err instanceof Error
                        ? err.message
                        : "Failed to create temperature observation";
            }
        } finally {
            creating = false;
        }
    };

    const toggleCreateForm = () => {
        showCreateForm = !showCreateForm;
        createError = null;
        createSuccess = false;
    };

    onMount(async () => {
        try {
            vitalObservations = await getVitals();
            loading = false;
        } catch (err) {
            error =
                err instanceof Error ? err.message : "Unknown error occurred";
            loading = false;
        }
    });
</script>

{#if loading}
    <div class="loading">Loading vital signs...</div>
{:else if error}
    <div class="error">Error loading vital sign observations: {error}</div>
{:else if vitalObservations}
    <div class="vitals-container">
        <div class="header-section">
            <h2>Vital Sign Observations</h2>
            <p class="subtitle">Most recent vital sign observations</p>
        </div>

        <div class="action-section">
            <button class="create-btn" on:click={toggleCreateForm}>
                {showCreateForm ? "Cancel" : "Add Temperature"}
            </button>
        </div>

        {#if createSuccess}
            <div class="success-message">
                Temperature observation created successfully!
            </div>
        {/if}

        {#if showCreateForm}
            <div class="create-form">
                <h3>Add New Temperature Reading</h3>

                {#if createError}
                    <div class="form-error">{createError}</div>
                {/if}

                <div class="form-group">
                    <label for="temperature">Temperature Value:</label>
                    <input
                        id="temperature"
                        type="number"
                        step="0.1"
                        bind:value={temperatureValue}
                        placeholder="Enter temperature"
                        disabled={creating}
                    />
                </div>

                <div class="form-group">
                    <div class="unit-display">
                        <span class="unit-label">Unit:</span>
                        <span class="unit-value">Celsius (°C)</span>
                    </div>
                </div>

                <div class="form-group">
                    <div class="datetime-info">
                        <span class="datetime-label">Date & Time:</span>
                        <div class="datetime-display">
                            {new Date().toLocaleString()} (current time)
                        </div>
                    </div>
                </div>

                <div class="form-actions">
                    <button
                        class="submit-btn"
                        on:click={createTemperatureObservation}
                        disabled={creating || !temperatureValue}
                    >
                        {creating
                            ? "Creating..."
                            : "Create Temperature Reading"}
                    </button>
                </div>
            </div>
        {/if}

        {#if getVitalsEntries(vitalObservations).length === 0}
            <div class="no-data">
                No vital sign observations found for this patient.
            </div>
        {:else}
            <div class="vitals-grid">
                {#each getVitalsEntries(vitalObservations).sort((a, b) => {
                    const obsA = a.resource as Observation;
                    const obsB = b.resource as Observation;
                    const dateA = new Date(obsA?.effectiveDateTime || "").getTime();
                    const dateB = new Date(obsB?.effectiveDateTime || "").getTime();
                    return dateB - dateA;
                }) as observation}
                    <div class="vital-card">
                        <h3 class="vital-name">
                            {(observation.resource as Observation)?.code
                                ?.text ?? "Unknown vital sign"}
                        </h3>
                        <p class="vital-date">
                            {formatDateTime(
                                (observation.resource as Observation)
                                    ?.effectiveDateTime,
                            )}
                        </p>
                        <p class="vital-value">
                            Value: {#if observation.resource}
                                {bpDisplay(observation.resource as Observation)}
                            {:else}
                                Not available
                            {/if}
                        </p>
                    </div>
                {/each}
            </div>
        {/if}
    </div>
{/if}

<style>
    .vitals-container {
        padding: 2rem;
        max-width: 1200px;
        margin: 0 auto;
    }

    .header-section {
        text-align: center;
        margin-bottom: 3rem;
        padding: 2rem 0;
    }

    .header-section h2 {
        color: var(--gray-900);
        margin-bottom: 0.75rem;
        font-size: 2.25rem;
        font-weight: 700;
        letter-spacing: -0.025em;
    }

    .subtitle {
        color: var(--gray-600);
        font-size: 1.125rem;
        margin: 0;
    }

    .action-section {
        display: flex;
        justify-content: center;
        gap: 1rem;
        margin-bottom: 2rem;
    }

    .create-btn {
        background: var(--primary-600);
        color: white;
        border: none;
        border-radius: 0.5rem;
        padding: 0.875rem 1.75rem;
        font-size: 0.875rem;
        font-weight: 600;
        cursor: pointer;
        transition: all 0.2s ease;
        box-shadow: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
    }

    .create-btn:hover {
        background: var(--primary-700);
        transform: translateY(-1px);
        box-shadow: 0 4px 12px 0 rgba(14, 165, 233, 0.15);
    }

    .create-form {
        background: white;
        border: 1px solid var(--gray-200);
        border-radius: 0.75rem;
        padding: 2rem;
        margin-bottom: 2rem;
        box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
    }

    .create-form h3 {
        color: var(--gray-900);
        margin-bottom: 1.5rem;
        font-size: 1.375rem;
        font-weight: 600;
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }



    .form-group {
        margin-bottom: 1.5rem;
    }

    .form-group label {
        display: block;
        color: var(--gray-700);
        font-weight: 500;
        margin-bottom: 0.5rem;
        font-size: 0.875rem;
    }

    .unit-display, .datetime-display {
        display: flex;
        align-items: center;
        gap: 0.75rem;
        padding: 0.875rem 1rem;
        background: var(--gray-50);
        border: 1px solid var(--gray-200);
        border-radius: 0.5rem;
        font-size: 0.875rem;
    }

    .unit-label, .datetime-label {
        color: var(--gray-600);
        font-weight: 500;
    }

    .unit-value {
        color: var(--gray-900);
        font-weight: 600;
    }

    .datetime-info {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
    }

    .datetime-display {
        color: var(--gray-700);
        font-style: italic;
        justify-content: flex-start;
    }

    .form-actions {
        margin-top: 2rem;
        display: flex;
        gap: 1rem;
    }

    .submit-btn {
        background: var(--success-600);
        color: white;
        border: none;
        border-radius: 0.5rem;
        padding: 0.875rem 1.75rem;
        font-size: 0.875rem;
        font-weight: 600;
        cursor: pointer;
        transition: all 0.2s ease;
        box-shadow: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
    }

    .submit-btn:hover:not(:disabled) {
        background: var(--success-700);
        transform: translateY(-1px);
        box-shadow: 0 4px 12px 0 rgba(34, 197, 94, 0.15);
    }

    .submit-btn:disabled {
        background: var(--gray-400);
        cursor: not-allowed;
        transform: none;
        box-shadow: none;
    }

    .success-message {
        background: var(--success-50);
        color: var(--success-700);
        border: 1px solid var(--success-200);
        border-radius: 0.5rem;
        padding: 1rem 1.25rem;
        margin-bottom: 1.5rem;
        font-weight: 500;
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }



    .form-error {
        background: var(--error-50);
        color: var(--error-700);
        border: 1px solid var(--error-200);
        border-radius: 0.5rem;
        padding: 1rem 1.25rem;
        margin-bottom: 1.5rem;
        font-size: 0.875rem;
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }



    .no-data {
        text-align: center;
        padding: 3rem 2rem;
        color: var(--gray-600);
        background: white;
        border: 1px solid var(--gray-200);
        border-radius: 0.75rem;
        font-size: 1.125rem;
    }



    .vitals-grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
        gap: 1.5rem;
    }

    .vital-card {
        background: white;
        border: 1px solid var(--gray-200);
        border-radius: 0.75rem;
        padding: 1.5rem;
        box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1), 0 1px 2px 0 rgba(0, 0, 0, 0.06);
        transition: all 0.2s ease;
        position: relative;
        overflow: hidden;
    }

    .vital-card::before {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        height: 4px;
        background: linear-gradient(90deg, var(--primary-500), var(--primary-600));
    }

    .vital-card:hover {
        box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
        transform: translateY(-2px);
    }

    .vital-name {
        color: var(--gray-900);
        font-size: 1.125rem;
        font-weight: 600;
        margin-bottom: 0.75rem;
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }



    .vital-date {
        color: var(--gray-500);
        font-size: 0.875rem;
        margin-bottom: 1rem;
        display: flex;
        align-items: center;
        gap: 0.375rem;
    }



    .vital-value {
        color: var(--gray-800);
        font-size: 1.125rem;
        font-weight: 600;
        background: var(--gray-50);
        padding: 0.75rem 1rem;
        border-radius: 0.5rem;
        border: 1px solid var(--gray-200);
    }

    @media (max-width: 768px) {
        .vitals-container {
            padding: 1rem;
        }
        
        .header-section {
            padding: 1rem 0;
            margin-bottom: 2rem;
        }
        
        .header-section h2 {
            font-size: 1.875rem;
        }
        
        .vitals-grid {
            grid-template-columns: 1fr;
            gap: 1rem;
        }
        
        .create-form {
            padding: 1.5rem;
        }
        
        .form-actions {
            flex-direction: column;
        }
    }
</style>
