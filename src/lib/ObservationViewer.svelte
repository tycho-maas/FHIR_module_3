<script lang="ts">
    // Import necessary libraries and types
    import axios from "axios"; // HTTP client for making API requests
    import type { Bundle, Observation } from "fhir/r4"; // FHIR R4 type definitions
    import { onMount } from "svelte"; // Svelte lifecycle function

    // Props passed from parent component - these are required inputs
    export let accessToken: string; // OAuth token for FHIR server authentication
    export let patientId: string; // ID of the patient whose vitals we're viewing
    export let baseUrl: string; // Base URL of the FHIR server

    // Component state variables for managing data and UI
    let vitalObservations: Bundle | undefined = undefined; // Stores the FHIR Bundle containing vital sign observations
    let loading = true; // Controls loading spinner display
    let error: string | null = null; // Stores any error messages from API calls
    let showCreateForm = false; // Controls visibility of the temperature creation form
    let creating = false; // Indicates if we're currently creating a new observation
    let createError: string | null = null; // Stores errors from creating observations
    let createSuccess = false; // Shows success message after creating an observation

    // Performance optimization: pagination
    let currentPage = 0;
    let itemsPerPage = 20;
    let displayedObservations: any[] = [];
    let allObservations: any[] = []; // Store all fetched observations
    let isLoadingMore = false;
    let hasMoreOnServer = true; // Track if server has more data
    let nextUrl: string | null = null; // FHIR pagination link

    // Form input variables
    let temperatureValue = ""; // Stores the temperature value entered by user

    // Cache for API responses
    let lastFetchTime = 0;
    const CACHE_DURATION = 30000; // 30 seconds cache

    /**
     * Fetches vital sign observations from the FHIR server with caching
     * Uses FHIR search parameters to filter and sort results
     */
    const getVitals = async (forceRefresh = false): Promise<Bundle> => {
        const now = Date.now();
        
        // Use cache if available and not expired (unless force refresh)
        if (!forceRefresh && vitalObservations && (now - lastFetchTime) < CACHE_DURATION) {
            return vitalObservations;
        }

        try {
            // Initial load with smaller count for faster response
            const response = await axios.get<Bundle>(
                `${baseUrl}/Observation?patient=${patientId}&category=vital-signs&_sort=-date&_count=30`,
                {
                    headers: {
                        Authorization: `Bearer ${accessToken}`,
                    },
                },
            );
            lastFetchTime = now;
            
            // Check for pagination links
            const bundle = response.data;
            nextUrl = bundle.link?.find(link => link.relation === 'next')?.url || null;
            hasMoreOnServer = !!nextUrl;
            
            return bundle;
        } catch (err) {
            throw new Error(`Failed to fetch vital signs: ${err}`);
        }
    };

    /**
     * Fetches more observations from the server using FHIR pagination
     */
    const fetchMoreFromServer = async (): Promise<Bundle | null> => {
        if (!nextUrl || isLoadingMore) return null;
        
        isLoadingMore = true;
        try {
            const response = await axios.get<Bundle>(nextUrl, {
                headers: {
                    Authorization: `Bearer ${accessToken}`,
                },
            });
            
            const bundle = response.data;
            // Update next URL for further pagination
            nextUrl = bundle.link?.find(link => link.relation === 'next')?.url || null;
            hasMoreOnServer = !!nextUrl;
            
            return bundle;
        } catch (err) {
            console.error('Failed to fetch more vitals:', err);
            return null;
        } finally {
            isLoadingMore = false;
        }
    };

    /**
     * Updates the displayed observations from all fetched data
     */
    const updateDisplayedObservations = () => {
        if (allObservations.length === 0 && vitalObservations) {
            // Initialize allObservations with first batch
            const entries = getVitalsEntries(vitalObservations);
            allObservations = entries.sort((a, b) => {
                const obsA = a.resource as Observation;
                const obsB = b.resource as Observation;
                const dateA = new Date(obsA?.effectiveDateTime || "").getTime();
                const dateB = new Date(obsB?.effectiveDateTime || "").getTime();
                return dateB - dateA;
            });
        }
        
        // Show items up to current page
        const endIndex = (currentPage + 1) * itemsPerPage;
        displayedObservations = allObservations.slice(0, endIndex);
    };

    const loadMore = async () => {
        // Check if we need more data from server
        const currentlyNeeded = (currentPage + 2) * itemsPerPage; // +2 because we're about to increment
        
        if (allObservations.length < currentlyNeeded && hasMoreOnServer && !isLoadingMore) {
            // Fetch more data from server
            const moreData = await fetchMoreFromServer();
            if (moreData) {
                const newEntries = getVitalsEntries(moreData);
                // Sort new entries and merge with existing
                const sortedNewEntries = newEntries.sort((a, b) => {
                    const obsA = a.resource as Observation;
                    const obsB = b.resource as Observation;
                    const dateA = new Date(obsA?.effectiveDateTime || "").getTime();
                    const dateB = new Date(obsB?.effectiveDateTime || "").getTime();
                    return dateB - dateA;
                });
                
                // Merge and re-sort all observations
                allObservations = [...allObservations, ...sortedNewEntries].sort((a, b) => {
                    const obsA = a.resource as Observation;
                    const obsB = b.resource as Observation;
                    const dateA = new Date(obsA?.effectiveDateTime || "").getTime();
                    const dateB = new Date(obsB?.effectiveDateTime || "").getTime();
                    return dateB - dateA;
                });
            }
        }
        
        currentPage++;
        updateDisplayedObservations();
    };

    const hasMore = () => {
        const currentlyDisplayed = (currentPage + 1) * itemsPerPage;
        return allObservations.length > currentlyDisplayed || hasMoreOnServer;
    };

    /**
     * Safely extracts the entry array from a FHIR Bundle
     * Returns empty array if no entries exist
     */
    const getVitalsEntries = (bundle: Bundle) => {
        return bundle.entry || []; // Use nullish coalescing to handle undefined entries
    };

    /**
     * Formats a date/time string for display
     * Converts ISO date strings to user-friendly local format
     */
    const formatDateTime = (dateTime: string | undefined): string => {
        if (!dateTime) return "Unknown date"; // Handle missing dates gracefully
        return new Date(dateTime).toLocaleString(); // Convert to local date/time format
    };

    /**
     * Formats observation values for display, handling different types of vital signs
     * Blood pressure observations have multiple components, while others have single values
     */
    const bpDisplay = (observation: Observation): string => {
        // Check if this is a multi-component observation (like blood pressure)
        if (observation.component && observation.component.length > 0) {
            // Look for systolic blood pressure component (LOINC code 8480-6)
            const systolic = observation.component.find((c) =>
                c.code?.coding?.some((coding) => coding.code === "8480-6"),
            );
            // Look for diastolic blood pressure component (LOINC code 8462-4)
            const diastolic = observation.component.find((c) =>
                c.code?.coding?.some((coding) => coding.code === "8462-4"),
            );

            // If both systolic and diastolic values exist, format as "120/80 mmHg"
            if (systolic?.valueQuantity && diastolic?.valueQuantity) {
                return `${systolic.valueQuantity.value}/${diastolic.valueQuantity.value} ${systolic.valueQuantity.unit || "mmHg"}`;
            }
        }

        // Handle single-value observations (temperature, heart rate, etc.)
        if (observation.valueQuantity) {
            return `${observation.valueQuantity.value} ${observation.valueQuantity.unit || ""}`;
        }

        // Handle string-based values (rare but possible)
        if (observation.valueString) {
            return observation.valueString;
        }

        // Fallback for observations without readable values
        return "Not available";
    };

    /**
     * Creates a new temperature observation and sends it to the FHIR server
     * Validates input, constructs FHIR-compliant observation, and handles errors
     */
    const createTemperatureObservation = async () => {
        // Validate that user entered a valid numeric temperature value
        if (!temperatureValue || isNaN(parseFloat(temperatureValue))) {
            createError = "Please enter a valid temperature value";
            return;
        }

        // Set UI state to show we're creating the observation
        creating = true;
        createError = null;
        createSuccess = false;

        try {
            // Construct a FHIR-compliant Observation resource for temperature
            const temperatureObservation: Observation = {
                resourceType: "Observation", // FHIR resource type
                status: "final", // Observation status (final = completed and verified)
                category: [
                    {
                        // Categorize this as a vital sign observation
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
                    // Use LOINC code 8331-1 for oral temperature
                    coding: [
                        {
                            system: "http://loinc.org", // LOINC coding system
                            code: "8331-1", // Specific code for oral temperature
                        },
                    ],
                    text: "Temperature Oral", // Human-readable description
                },
                subject: {
                    // Link this observation to the specific patient
                    reference: `Patient/${patientId}`,
                },
                effectiveDateTime: new Date().toISOString(), // When the measurement was taken (now)
                valueQuantity: {
                    // The actual temperature measurement
                    value: parseFloat(temperatureValue), // Convert string input to number
                    unit: "degC", // Display unit
                    system: "http://unitsofmeasure.org", // UCUM units system
                    code: "Cel", // UCUM code for Celsius
                },
            };

            // Debug logging to help troubleshoot API calls
            console.log(
                "Creating temperature observation:",
                JSON.stringify(temperatureObservation, null, 2),
            );
            console.log("POST URL:", `${baseUrl}/Observation`);
            console.log("Headers:", {
                Authorization: `Bearer ${accessToken}`,
                "Content-Type": "application/fhir+json",
            });

            // Send the observation to the FHIR server via HTTP POST
            const response = await axios.post(
                `${baseUrl}/Observation`,
                temperatureObservation,
                {
                    headers: {
                        Authorization: `Bearer ${accessToken}`, // Authentication
                        "Content-Type": "application/fhir+json", // FHIR-specific content type
                    },
                },
            );

            // Log response details for debugging
            console.log("POST response status:", response.status);
            console.log("POST response headers:", response.headers);
            console.log(
                "POST response data:",
                JSON.stringify(response.data, null, 2),
            );
            console.log("Created observation ID:", response.data?.id);
            console.log("Location header:", response.headers?.location);

            // Check if we got the created resource back in the response
            if (response.data) {
                console.log("Full created observation:", response.data);
            } else {
                console.log(
                    "No response body - check Location header for created resource URL",
                );
            }

            // Optimistic update: add the new observation immediately
            const newEntry = {
                resource: {
                    ...temperatureObservation,
                    id: response.data?.id || `temp-${Date.now()}`,
                    effectiveDateTime: temperatureObservation.effectiveDateTime
                }
            };

            // Add to the beginning of all observations for immediate display
            allObservations = [newEntry, ...allObservations];
            
            // Update the bundle
            if (vitalObservations) {
                vitalObservations.entry = [newEntry, ...(vitalObservations.entry || [])];
            }
            
            // Refresh displayed observations
            updateDisplayedObservations();

            // Update UI to show success and hide the form
            createSuccess = true;
            showCreateForm = false;
            temperatureValue = "";

            // Background refresh to sync with server (no loading state)
            setTimeout(() => {
                getVitals(true).then(bundle => {
                    vitalObservations = bundle;
                    allObservations = []; // Reset to force re-initialization
                    currentPage = 0; // Reset pagination
                    updateDisplayedObservations();
                });
            }, 1000);
        } catch (err) {
            // Handle errors during observation creation
            console.error("Error creating temperature observation:", err);
            if (axios.isAxiosError(err)) {
                // Extract meaningful error information from HTTP response
                console.error("Response status:", err.response?.status);
                console.error("Response data:", err.response?.data);
                createError = `HTTP ${err.response?.status}: ${err.response?.data?.issue?.[0]?.diagnostics || err.message}`;
            } else {
                // Handle non-HTTP errors
                createError =
                    err instanceof Error
                        ? err.message
                        : "Failed to create temperature observation";
            }
        } finally {
            // Always reset the creating flag, regardless of success or failure
            creating = false;
        }
    };

    /**
     * Toggles the visibility of the temperature creation form
     * Also resets any error or success messages
     */
    const toggleCreateForm = () => {
        showCreateForm = !showCreateForm; // Toggle form visibility
        createError = null; // Clear any previous errors
        createSuccess = false; // Clear success message
    };

    // Reactive statement to update displayed observations when data changes
    $: if (vitalObservations) {
        updateDisplayedObservations();
    }

    /**
     * Svelte lifecycle function that runs when the component is first mounted
     * Automatically fetches vital signs data when the component loads
     */
    onMount(async () => {
        try {
            // Fetch initial vital signs data
            vitalObservations = await getVitals();
            loading = false; // Hide loading spinner
        } catch (err) {
            // Handle errors during initial data fetch
            error =
                err instanceof Error ? err.message : "Unknown error occurred";
            loading = false; // Hide loading spinner even on error
        }
    });
</script>

<!-- 
    Svelte template with conditional rendering based on component state
    Shows different UI states: loading, error, or main content
-->

<!-- Show loading skeleton while fetching data -->
{#if loading}
    <div class="vitals-container">
        <div class="header-section">
            <h2>Vital Sign Observations</h2>
            <p class="subtitle">Most recent vital sign observations</p>
        </div>
        
        <div class="action-section">
            <div class="skeleton skeleton-button"></div>
        </div>
        
        <div class="vitals-grid">
            {#each Array(6) as _}
                <div class="vital-card skeleton-card">
                    <div class="skeleton skeleton-title"></div>
                    <div class="skeleton skeleton-date"></div>
                    <div class="skeleton skeleton-value"></div>
                </div>
            {/each}
        </div>
    </div>

    <!-- Show error message if data fetch failed -->
{:else if error}
    <div class="error">Error loading vital sign observations: {error}</div>

    <!-- Main content area - only shown when data is successfully loaded -->
{:else if vitalObservations}
    <div class="vitals-container">
        <!-- Page header with title and description -->
        <div class="header-section">
            <h2>Vital Sign Observations</h2>
            <p class="subtitle">Most recent vital sign observations</p>
        </div>

        <!-- Action buttons section -->
        <div class="action-section">
            <!-- Toggle button to show/hide temperature creation form -->
            <button class="create-btn" on:click={toggleCreateForm}>
                {showCreateForm ? "Cancel" : "Add Temperature"}
            </button>
        </div>

        <!-- Success message shown after successfully creating a temperature observation -->
        {#if createSuccess}
            <div class="success-message">
                Temperature observation created successfully!
            </div>
        {/if}

        <!-- Temperature creation form - only visible when showCreateForm is true -->
        {#if showCreateForm}
            <div class="create-form">
                <h3>Add New Temperature Reading</h3>

                <!-- Show error message if there was a problem creating the observation -->
                {#if createError}
                    <div class="form-error">{createError}</div>
                {/if}

                <!-- Temperature input field -->
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

                <!-- Display the unit (Celsius) - read-only information -->
                <div class="form-group">
                    <div class="unit-display">
                        <span class="unit-label">Unit:</span>
                        <span class="unit-value">Celsius (°C)</span>
                    </div>
                </div>

                <!-- Show current date/time that will be used for the observation -->
                <div class="form-group">
                    <div class="datetime-info">
                        <span class="datetime-label">Date & Time:</span>
                        <div class="datetime-display">
                            {new Date().toLocaleString()} (current time)
                        </div>
                    </div>
                </div>

                <!-- Form submission button -->
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

        <!-- Check if there are any vital sign observations to display -->
        {#if allObservations.length === 0}
            <!-- Show message when no vital signs are found -->
            <div class="no-data">
                No vital sign observations found for this patient.
            </div>
        {:else}
            <!-- Grid layout for displaying vital sign cards -->
            <div class="vitals-grid">
                <!-- Loop through paginated observations (already sorted) -->
                {#each displayedObservations as observation}
                    <!-- Individual card for each vital sign observation -->
                    <div class="vital-card">
                        <!-- Display the name/type of vital sign (e.g., "Temperature Oral") -->
                        <h3 class="vital-name">
                            {(observation.resource as Observation)?.code
                                ?.text ?? "Unknown vital sign"}
                        </h3>
                        <!-- Display when the observation was taken -->
                        <p class="vital-date">
                            {formatDateTime(
                                (observation.resource as Observation)
                                    ?.effectiveDateTime,
                            )}
                        </p>
                        <!-- Display the actual measurement value -->
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
            
            <!-- Load more button for pagination -->
            {#if hasMore()}
                <div class="load-more-section">
                    <button class="load-more-btn" on:click={loadMore} disabled={isLoadingMore}>
                        {#if isLoadingMore}
                            <span class="loading-spinner"></span>
                            Loading More...
                        {:else}
                            Load More Vitals
                        {/if}
                    </button>
                </div>
            {/if}
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
        box-shadow:
            0 4px 6px -1px rgba(0, 0, 0, 0.1),
            0 2px 4px -1px rgba(0, 0, 0, 0.06);
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

    .unit-display,
    .datetime-display {
        display: flex;
        align-items: center;
        gap: 0.75rem;
        padding: 0.875rem 1rem;
        background: var(--gray-50);
        border: 1px solid var(--gray-200);
        border-radius: 0.5rem;
        font-size: 0.875rem;
    }

    .unit-label,
    .datetime-label {
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
        box-shadow:
            0 1px 3px 0 rgba(0, 0, 0, 0.1),
            0 1px 2px 0 rgba(0, 0, 0, 0.06);
        transition: all 0.2s ease;
        position: relative;
        overflow: hidden;
    }

    .vital-card::before {
        content: "";
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        height: 4px;
        background: linear-gradient(
            90deg,
            var(--primary-500),
            var(--primary-600)
        );
    }

    .vital-card:hover {
        box-shadow:
            0 10px 25px -5px rgba(0, 0, 0, 0.1),
            0 10px 10px -5px rgba(0, 0, 0, 0.04);
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

    .load-more-section {
        display: flex;
        justify-content: center;
        margin-top: 2rem;
        padding: 1rem;
    }

    .load-more-btn {
        background: var(--primary-100);
        color: var(--primary-700);
        border: 1px solid var(--primary-200);
        border-radius: 0.5rem;
        padding: 0.875rem 1.75rem;
        font-size: 0.875rem;
        font-weight: 600;
        cursor: pointer;
        transition: all 0.2s ease;
        box-shadow: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
    }

    .load-more-btn:hover:not(:disabled) {
        background: var(--primary-200);
        transform: translateY(-1px);
        box-shadow: 0 4px 12px 0 rgba(14, 165, 233, 0.15);
    }

    .load-more-btn:disabled {
        opacity: 0.6;
        cursor: not-allowed;
        transform: none;
    }

    .loading-spinner {
        display: inline-block;
        width: 1rem;
        height: 1rem;
        border: 2px solid transparent;
        border-top: 2px solid currentColor;
        border-radius: 50%;
        animation: spin 1s linear infinite;
        margin-right: 0.5rem;
    }

    /* Skeleton loading styles */
    .skeleton {
        background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
        background-size: 200% 100%;
        animation: shimmer 1.5s infinite;
        border-radius: 0.5rem;
    }

    .skeleton-button {
        height: 2.5rem;
        width: 150px;
        border-radius: 0.5rem;
    }

    .skeleton-card {
        background: white;
        border: 1px solid var(--gray-200);
        border-radius: 0.75rem;
        padding: 1.5rem;
        box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1), 0 1px 2px 0 rgba(0, 0, 0, 0.06);
    }

    .skeleton-title {
        height: 1.5rem;
        width: 80%;
        margin-bottom: 0.75rem;
    }

    .skeleton-date {
        height: 1rem;
        width: 60%;
        margin-bottom: 1rem;
    }

    .skeleton-value {
        height: 2rem;
        width: 100%;
        border-radius: 0.5rem;
    }

    @keyframes shimmer {
        0% {
            background-position: -200% 0;
        }
        100% {
            background-position: 200% 0;
        }
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

        .load-more-btn {
            width: 100%;
        }

        .skeleton-button {
            width: 100%;
        }
    }
</style>
