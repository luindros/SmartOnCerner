<script lang="ts">
    import { onMount } from 'svelte';
    import axios from 'axios';
    import PatientBanner from './PatientBanner.svelte';
    import PatientVital from './PatientVital.svelte';
  
    // Declare variables to hold the necessary data
    let iss: string | null = null; // Variable for issuer
    let issStored: string | null = null; 
    let launch: string | null = null; // Variable for launch
    let authorizationEndpoint: string | null = null; // Variable for authorization endpoint
    let tokenEndpoint: string | null = null; // Variable for token endpoint
    let authUrl: string | null = null; // Variable for constructed authorization URL
    interface tokenResponseType {
        need_patient_banner: boolean;
        id_token: string;
        smart_style_url: string;
        token_type: string;
        access_token: string;
        patient: string;
        scope: string;
        expires_in: number;
    }
    let tokenResponse: tokenResponseType | undefined;
  
    const LOCAL_STORAGE_TOKEN_RESPONSE = 'LOCAL_STORAGE_TOKEN_RESPONSE'; // Key for local storage
    const LOCAL_STORAGE_ISS = 'LOCAL_STORAGE_ISS'; // Key for local storage
    const LOCAL_STORAGE_TOKEN_ENDPOINT = 'LOCAL_STORAGE_TOKEN_ENDPOINT'; // Key for token endpoint
    const clientId = 'cf372d46-fa49-45a5-b54a-d9f7e3173631'; // Replace with your actual client ID
    const redirectUri = 'http://localhost:5173'; // Replace with your actual redirect URI
  
    
   
    // Function to save the token to local storage
    function saveToken(token: any) {
      localStorage.setItem(LOCAL_STORAGE_TOKEN_RESPONSE, JSON.stringify(token));
    }
  
    // Function to save the token endpoint to local storage
    function saveTokenEndPoint(endpoint: string | null) {
      if (endpoint) {
        localStorage.setItem(LOCAL_STORAGE_TOKEN_ENDPOINT, endpoint); // Store the token endpoint in local storage
      } else {
        console.warn('Token endpoint is null, not saving to local storage.'); // Log a warning if null
      }
    }

    function saveISS(iss: string | null) {
      if (iss) {
        localStorage.setItem(LOCAL_STORAGE_ISS, iss); // Store the token endpoint in local storage
      } else {
        console.warn('iss is null, not saving to local storage.'); // Log a warning if null
      }
    }
  
  
    /// Function to construct the authorization URL
    function constructAuthURL(endpoint: string): string {
      const params = new URLSearchParams({
        response_type: 'code',
        client_id: clientId,
        redirect_uri: redirectUri,
        scope: 'openid fhirUser launch offline_access user/Patient.read user/Observation.read user/Observation.write',
        aud: iss || '', // Provide a default value if iss is null
        launch: launch || ''
      });
      return `${endpoint}?${params.toString()}`; // Return the full URL
    }
  
    // Function to call the token endpoint
    async function callTokenEndpoint(code: string): Promise<tokenResponseType | null> {
      tokenEndpoint = localStorage.getItem(LOCAL_STORAGE_TOKEN_ENDPOINT); // Retrieve token endpoint from local storage
      if (!tokenEndpoint) {
        console.error('Token endpoint is not defined.');
        return null; // Return null if tokenEndpoint is not available
      }
  
      try {
        const response = await axios.post(tokenEndpoint, {
          grant_type: 'authorization_code',
          code: code,
          redirect_uri: redirectUri,
          client_id: clientId,
          client_secret: 'YOUR_CLIENT_SECRET' // Replace with your actual client secret
        }, {
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded'
          }
        });
        return response.data as tokenResponseType; // Ensure the response is cast to tokenResponseType
      } catch (error) {
        console.error('Error fetching token:', error);
        return null; // Return null on error
      }
    }
  
    // Lifecycle method to run on component mount
    onMount(async () => {
   // localStorage.clear();
      const urlParams = new URLSearchParams(window.location.search);
      iss = decodeURIComponent(urlParams.get('iss') || ''); // Get and decode iss
      launch = decodeURIComponent(urlParams.get('launch') || ''); // Get and decode launch
      const code = decodeURIComponent(urlParams.get('code') || ''); // Get and decode code
  
      console.log({ iss, launch }); // Log the values
      
      // Check local storage for the token response
      saveISS(iss); 
      issStored=localStorage.getItem(LOCAL_STORAGE_ISS) //store iss to local storage

      const storedTokenResponse = localStorage.getItem(LOCAL_STORAGE_TOKEN_RESPONSE);
      if (storedTokenResponse) {
        tokenResponse = JSON.parse(storedTokenResponse); // Parse the token from JSON
        console.log('Token retrieved:', tokenResponse); // Log the token
      } else {
        console.error('Token response not found in local storage.');
      }
  
      // If a code is present, call the token endpoint
      if (code) {
        const tokenCerner = await callTokenEndpoint(code);
        if (tokenCerner) {
          tokenResponse = tokenCerner; // Set the token response
          saveToken(tokenResponse); // Save the token to local storage
        }
      } else {
        try {
          // Fetch the smart configuration
          const response = await axios.get(`${iss}/.well-known/smart-configuration`);
          const smartConfig = response.data; // Store the response data
  
          // Extract authorization and token endpoints
          authorizationEndpoint = smartConfig.authorization_endpoint || null;
          tokenEndpoint = smartConfig.token_endpoint || null;
  
          // Save the token endpoint to local storage
          saveTokenEndPoint(tokenEndpoint); // Store the token endpoint in local storage
  
          console.log({ authorizationEndpoint, tokenEndpoint }); // Log the endpoints
  
          // Construct the authorization URL
          if (authorizationEndpoint) {
            authUrl = constructAuthURL(authorizationEndpoint); // Call the method to construct the URL
            console.log('Authorization URL:', authUrl); // Log the constructed URL
            window.location.href = authUrl; // Redirect to the authorization URL
          }
        } catch (error) {
          console.error('Error fetching smart configuration:', error);
        }
      }
    });
  </script>

{#if !tokenResponse}
  Loading...
 {:else}
 {#if tokenResponse?.need_patient_banner}
    <PatientBanner baseURL={issStored || ''} accessToken={tokenResponse.access_token} patient={tokenResponse.patient}></PatientBanner>
    <PatientVital baseURL={issStored || ''} accessToken={tokenResponse.access_token}  patient={tokenResponse.patient}></PatientVital>
 {/if} 
 {/if}   