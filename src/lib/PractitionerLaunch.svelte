<script lang="ts">
    import axios from "axios";
    import { onMount } from "svelte";
    import PatientBanner from "./PatientBanner.svelte";
    import ObservationViewer from "./ObservationViewer.svelte";

    let baseUrl: string;
    let launch: string;
    let redirectUrl = "http://localhost:5173"

    interface tokenResponseType {
        "access_token": string,
        "patient": string,
        "scope": string,
        "need_patient_banner": boolean,
        "id_token": string
        "smart_style_url": string,
        "token_type": string,
        "expires_in": number
    }

    let token: tokenResponseType | undefined = undefined


    let client_id = "b8ee6437-99b8-471b-8b98-c89ca1451150"

    const constructAuthUrl = (authorizationEndpoint: string, launch: string, iss: string) => {
        const url = new URL(authorizationEndpoint);
        url.searchParams.set(
            "client_id",client_id);
        url.searchParams.set("redirect_uri", redirectUrl);
        url.searchParams.set(
            "scope",
            "openid fhirUser launch user/Observation.read user/Observation.write user/Patient.read",
        );
        url.searchParams.set("response_type", "code");
        url.searchParams.set("aud", iss);

        url.searchParams.set("launch", launch);

        return url.href;
    };

    const clearAuthData = () => {
        localStorage.removeItem("token");
        localStorage.removeItem("iss");
        localStorage.removeItem("tokenEndpoint");
        localStorage.removeItem("currentLaunch");
        token = undefined;
    };

    onMount(async () => {
        const launchUrl = new URL(window.location.href);
        const issParam = launchUrl.searchParams.get("iss");
        const launchParam = launchUrl.searchParams.get("launch");

        const code = launchUrl.searchParams.get("code");

        // If we have new launch parameters, this is a fresh launch - clear old data
        if (issParam && launchParam) {
            // Store the current launch parameters to detect new launches
            const storedLaunch = localStorage.getItem("currentLaunch");
            const currentLaunchKey = `${issParam}:${launchParam}`;
            
            if (storedLaunch && storedLaunch !== currentLaunchKey) {
                // Different launch parameters mean new patient/session, clear old data
                clearAuthData();
            }
            
            // Store the current launch for future comparison
            localStorage.setItem("currentLaunch", currentLaunchKey);
        }

        const tokenJSON = localStorage.getItem("token")
        const issLocalStorage = localStorage.getItem("iss")

        if (issLocalStorage){
            baseUrl = issLocalStorage
        }

        if (tokenJSON){
            token = JSON.parse(tokenJSON)
        }

        if (token){
            // TODO validate token expiration
            return
        }

        if (code){
            const tokenFromCerner = await makeTokenRequest(code); 
            console.log({tokenFromCerner})
            token = tokenFromCerner
            localStorage.setItem("token", JSON.stringify(token));

            return
        }

        if (!issParam || !launchParam) {
            throw new Error("ISS or Launch parameter is missing.");
        }
        const iss = issParam;
        localStorage.setItem("iss", issParam)

        const launch = launchParam;

        const smartConfigurationResponse = await axios.get(
            `${iss}/.well-known/smart-configuration`,
        );
        const smartConfiguration = smartConfigurationResponse.data;
        const authorizationEndpoint =
            smartConfiguration.authorization_endpoint as string;
        const tokenEndpoint = smartConfiguration.token_endpoint as string;

        // store tokenEndpoint for future use
        localStorage.setItem("tokenEndpoint", tokenEndpoint);

        const redirectUrl = constructAuthUrl(authorizationEndpoint, launch, iss);

        console.log({ iss, launch, smartConfiguration });

        window.location.href = redirectUrl;
    });


    const makeTokenRequest = async (code: string) => {

        const tokenEndpoint = localStorage.getItem("tokenEndpoint");

        if (!tokenEndpoint) {
            throw new Error("Token endpoint is not in local storage.");
        }

        // Make token request

const tokenResponse = await axios.post(tokenEndpoint, new URLSearchParams({
            "grant_type": "authorization_code",
            "code": code,
            "redirect_uri": redirectUrl,
            "client_id": client_id
        }), {
            headers: {
                "Content-Type": "application/x-www-form-urlencoded"
            }
        });


        // Don't remove tokenEndpoint here - keep it for potential reloads
        // localStorage.removeItem("tokenEndpoint");

        return tokenResponse.data
    }
</script>
{#if !token}
    <div class="loading">Loading...</div>
{:else}
    {#if token?.need_patient_banner}
        <PatientBanner {baseUrl} accessToken={token.access_token} patient={token.patient} />
    {/if}
    
    <div class="main-content">
        <ObservationViewer accessToken={token.access_token} patientId={token.patient} {baseUrl} />
    </div>
{/if}

<style>
    .loading {
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        height: 100vh;
        background: linear-gradient(135deg, var(--gray-50) 0%, var(--primary-50) 100%);
        gap: 1.5rem;
    }

    .loading::before {
        content: '';
        width: 3rem;
        height: 3rem;
        border: 4px solid var(--gray-200);
        border-top: 4px solid var(--primary-600);
        border-radius: 50%;
        animation: spin 1s linear infinite;
    }

    .loading::after {
        content: 'Loading your healthcare data...';
        font-size: 1.125rem;
        color: var(--gray-600);
        font-weight: 500;
    }

    @keyframes spin {
        0% { transform: rotate(0deg); }
        100% { transform: rotate(360deg); }
    }

    .main-content {
        margin-top: 100px; /* Account for fixed patient banner */
        min-height: calc(100vh - 100px);
        background: linear-gradient(135deg, var(--gray-50) 0%, var(--primary-50) 100%);
    }

    @media (max-width: 768px) {
        .main-content {
            margin-top: 80px;
            min-height: calc(100vh - 80px);
        }
    }
</style>
