<script lang="ts">
    import axios from "axios";
    import type { Patient } from "fhir/r4";
    import { onMount } from "svelte";

    export let accessToken: string;
    export let baseUrl: string;
    export let patient: string;

    let patientResource: Patient | undefined = undefined;
    let loading = true;

    // Simple cache for patient data
    let patientCache: { [key: string]: { data: Patient; timestamp: number } } = {};
    const CACHE_DURATION = 300000; // 5 minutes cache for patient data

    const getPatientData = async (): Promise<Patient> => {
        const cacheKey = `${baseUrl}-${patient}`;
        const now = Date.now();
        
        // Check cache first
        if (patientCache[cacheKey] && (now - patientCache[cacheKey].timestamp) < CACHE_DURATION) {
            return patientCache[cacheKey].data;
        }

        const response = await axios.get<Patient>(
            `${baseUrl}/Patient/${patient}`,
            {
                headers: {
                    Authorization: `Bearer ${accessToken}`,
                },
            },
        );

        // Cache the result
        patientCache[cacheKey] = {
            data: response.data,
            timestamp: now
        };

        return response.data;
    };

    onMount(async () => {
        try {
            patientResource = await getPatientData();
        } catch (error) {
            console.error('Failed to load patient data:', error);
        } finally {
            loading = false;
        }
    });
</script>

<!-- <pre>
    {JSON.stringify(patientResource,null,2)}
</pre> -->

<div class="patient-banner">
    {#if patientResource}
        <div class="patient-info">
            <span class="patient-name"
                >{patientResource?.name?.[0]?.given?.[0]}
                {patientResource?.name?.[0]?.family}</span
            >
            <span class="patient-detail">Gender: {patientResource?.gender}</span
            >
            <span class="patient-detail">DOB: {patientResource?.birthDate}</span
            >
        </div>
    {:else}
        <div class="patient-info">
            <div class="skeleton skeleton-name"></div>
            <div class="skeleton skeleton-detail"></div>
            <div class="skeleton skeleton-detail"></div>
        </div>
    {/if}
</div>

<style>
    .patient-banner {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        background: linear-gradient(
            135deg,
            var(--primary-600) 0%,
            var(--primary-700) 100%
        );
        border-bottom: 1px solid var(--primary-800);
        padding: 1.25rem 2rem;
        z-index: 1000;
        box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
        backdrop-filter: blur(10px);
    }

    .patient-info {
        display: flex;
        align-items: center;
        gap: 1.5rem;
        flex-wrap: wrap;
        max-width: 1200px;
        margin: 0 auto;
    }

    .patient-name {
        font-size: 1.375rem;
        font-weight: 600;
        color: white;
        text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
        letter-spacing: -0.025em;
    }

    .patient-detail {
        color: rgba(255, 255, 255, 0.95);
        font-size: 0.875rem;
        font-weight: 500;
        background: rgba(255, 255, 255, 0.15);
        padding: 0.375rem 0.875rem;
        border-radius: 1rem;
        backdrop-filter: blur(10px);
        border: 1px solid rgba(255, 255, 255, 0.1);
        transition: all 0.2s ease;
    }

    .patient-detail:hover {
        background: rgba(255, 255, 255, 0.2);
        transform: translateY(-1px);
    }

    .patient-banner p {
        margin: 0;
        color: white;
        font-size: 1rem;
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }

    .skeleton {
        background: linear-gradient(90deg, rgba(255,255,255,0.1) 25%, rgba(255,255,255,0.2) 50%, rgba(255,255,255,0.1) 75%);
        background-size: 200% 100%;
        animation: shimmer 1.5s infinite;
        border-radius: 0.5rem;
    }

    .skeleton-name {
        height: 1.5rem;
        width: 200px;
    }

    .skeleton-detail {
        height: 1.25rem;
        width: 120px;
        border-radius: 1rem;
    }

    @keyframes shimmer {
        0% {
            background-position: -200% 0;
        }
        100% {
            background-position: 200% 0;
        }
    }

    @keyframes spin {
        0% {
            transform: rotate(0deg);
        }
        100% {
            transform: rotate(360deg);
        }
    }

    @media (max-width: 768px) {
        .patient-banner {
            padding: 1rem 1.5rem;
        }

        .patient-info {
            gap: 1rem;
        }

        .patient-name {
            font-size: 1.25rem;
        }

        .patient-detail {
            font-size: 0.8125rem;
            padding: 0.25rem 0.75rem;
        }
    }
</style>
