```
package com.example.checkordering.client;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpHeaders;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

/**
 * Gateway for executing REST API calls to external services.
 * This class encapsulates the details of making HTTP requests using RestClient.
 */
@Component
@Slf4j
@RequiredArgsConstructor
public class RestApiGateway {

    private final RestClient restClient;

    /**
     * Executes a GET request to the specified URL with authentication.
     *
     * @param url          The URL to send the GET request to.
     * @param token        The authentication token to be included in the request header.
     * @param responseType The expected response type.
     * @param <T>          The type of the response body.
     * @return The response body converted to the specified type.
     */
    public <T> T executeGetRequest(String url, String token, Class<T> responseType) {
        log.info("Executing GET request to URL: {}", url);

        return restClient.get()
                .uri(url)
                .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                .retrieve()
                .body(responseType);
    }
}

```

```
package com.example.checkordering.service;

import com.example.checkordering.client.OktaClient;
import com.example.checkordering.client.RestApiGateway;
import com.example.checkordering.exception.AccountServiceException;
import com.example.checkordering.exception.PartyServiceException;
import com.example.checkordering.model.AccountDetails;
import com.example.checkordering.model.AccountRole;
import com.example.checkordering.model.Party;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.util.Assert;
import org.springframework.web.util.UriComponentsBuilder;

import jakarta.annotation.PostConstruct;
import java.util.HashMap;
import java.util.Map;

/**
 * Service for validating accounts and retrieving account details.
 */
@Service
@Slf4j
@RequiredArgsConstructor
public class AccountValidationService {

    private final RestApiGateway restApiGateway;
    private final OktaClient oktaClient;

    @Value("${client.party}")
    private String partyClient;

    @Value("${client.account}")
    private String accountClient;

    @Value("${scopes.party.read}")
    private String partyReadScope;

    @Value("${scopes.limited.accts.b2c.read}")
    private String limitedAcctsB2cReadScope;

    @Value("${party-service.base-url}")
    private String partyServiceBaseUrl;

    @Value("${account-service.base-url}")
    private String accountServiceBaseUrl;

    @Value("${api.party-service.ecif}")
    private String partyServiceEcifPath;

    @Value("${api.party-service.account-role}")
    private String partyServiceAccountRolePath;

    @Value("${api.account-details}")
    private String accountDetailsPath;

    /**
     * Validates the properties after construction.
     */
    @PostConstruct
    private void validateProperties() {
        Assert.notNull(partyServiceBaseUrl, "partyServiceBaseUrl must not be null");
        Assert.notNull(accountServiceBaseUrl, "accountServiceBaseUrl must not be null");
        Assert.notNull(partyServiceEcifPath, "partyServiceEcifPath must not be null");
        Assert.notNull(partyServiceAccountRolePath, "partyServiceAccountRolePath must not be null");
        Assert.notNull(accountDetailsPath, "accountDetailsPath must not be null");
    }

    /**
     * Validates an account based on the provided information.
     *
     * @param uuid   The UUID of the request.
     * @param acctId The account ID.
     * @param ecifId The ECIF ID.
     * @return true if the account is valid, false otherwise.
     * @throws Exception if an error occurs during validation.
     */
    public boolean validateAccount(String uuid, String acctId, String ecifId) throws Exception {
        log.info("Validating account for UUID: {} and Account ID: {}", uuid, acctId);
        String token = oktaClient.getToken(partyClient, partyReadScope);

        Party party = getParty(ecifId, token);
        AccountRole accountRole = getAccountRole(acctId, token);

        boolean isValid = accountRole.getAccountRole().stream()
                .peek(role -> log.info("Role Party Domain ID: {}", role.getParty().getPartyDomainId()))
                .anyMatch(role -> {
                    String rolePartyDomainId = role.getParty().getPartyDomainId();
                    String partyDomainId = party.getParty().getPartyDomainId();
                    log.info("Comparing Role Party Domain ID: {} with Party Domain ID: {}", rolePartyDomainId,
                            partyDomainId);
                    return rolePartyDomainId.equals(partyDomainId);
                });

        log.info("Account validation result for UUID: {} and Account ID: {} isValid: {}", uuid, acctId, isValid);
        return isValid;
    }

    /**
     * Retrieves account details for the given account ID and affiliate.
     *
     * @param acctId    The account ID.
     * @param affiliate The affiliate information.
     * @return AccountDetails object containing the account information.
     * @throws AccountServiceException if an error occurs while fetching account details.
     */
    public AccountDetails getAccountDetails(String acctId, String affiliate) {
        try {
            log.info("Fetching account details for Account ID: {} and Affiliate: {}", acctId, affiliate);
            String token = oktaClient.getToken(accountClient, limitedAcctsB2cReadScope);

            String url = buildUrl(accountServiceBaseUrl, accountDetailsPath, acctId);

            Map<String, String> queryParams = new HashMap<>();
            queryParams.put("affiliate", affiliate);
            url = addQueryParams(url, queryParams);

            log.info("Final URL with query params: {}", url);
            AccountDetails accountDetails = restApiGateway.executeGetRequest(url, token, AccountDetails.class);
            log.info("Successfully retrieved account details for Account ID: {} and Affiliate: {}", acctId, affiliate);
            return accountDetails;
        } catch (Exception e) {
            log.error("Error fetching account details for Account ID: {} and Affiliate: {}", acctId, affiliate, e);
            throw new AccountServiceException("Failed to fetch account details", e);
        }
    }

    /**
     * Retrieves party information for the given ECIF ID.
     *
     * @param ecifId The ECIF ID.
     * @param token  The authentication token.
     * @return Party object containing the party information.
     * @throws PartyServiceException if an error occurs while fetching party information.
     */
    protected Party getParty(String ecifId, String token) {
        try {
            log.info("Fetching party information for ECIFID: {}", ecifId);
            String url = buildUrl(partyServiceBaseUrl, partyServiceEcifPath, ecifId);

            Party party = restApiGateway.executeGetRequest(url, token, Party.class);
            log.info("Successfully retrieved party information for ecifId: {}", ecifId);
            return party;
        } catch (Exception e) {
            log.error("Error fetching party information for ecifId: {}", ecifId, e);
            throw new PartyServiceException("Failed to fetch party information", e);
        }
    }

    /**
     * Retrieves account role information for the given account ID.
     *
     * @param acctId The account ID.
     * @param token  The authentication token.
     * @return AccountRole object containing the account role information.
     * @throws PartyServiceException if an error occurs while fetching account role information.
     */
    protected AccountRole getAccountRole(String acctId, String token) {
        try {
            log.info("Fetching account role for Account ID: {}", acctId);
            String url = buildUrl(partyServiceBaseUrl, partyServiceAccountRolePath, acctId);

            AccountRole accountRole = restApiGateway.executeGetRequest(url, token, AccountRole.class);
            log.info("Successfully retrieved account role for Account ID: {}", acctId);
            return accountRole;
        } catch (Exception e) {
            log.error("Error fetching account role for Account ID: {}", acctId, e);
            throw new PartyServiceException("Failed to fetch account role", e);
        }
    }

    /**
     * Adds query parameters to the given URL.
     *
     * @param url         The base URL.
     * @param queryParams Map of query parameters to add.
     * @return The URL with added query parameters.
     */
    private String addQueryParams(String url, Map<String, String> queryParams) {
        UriComponentsBuilder uriBuilder = UriComponentsBuilder.fromUriString(url);
        queryParams.forEach(uriBuilder::queryParam);
        return uriBuilder.encode().toUriString();
    }

    /**
     * Builds a URL from the given base URL, path, and path variables.
     *
     * @param baseUrl       The base URL.
     * @param path          The path to append to the base URL.
     * @param pathVariables Variables to be inserted into the path.
     * @return The complete URL.
     */
    private String buildUrl(String baseUrl, String path, Object... pathVariables) {
        return UriComponentsBuilder.fromUriString(baseUrl)
                .path(path)
                .buildAndExpand(pathVariables)
                .encode()
                .toUriString();
    }
}
```


```
package com.example.checkordering.service;

import com.example.checkordering.client.OktaClient;
import com.example.checkordering.client.RestApiGateway;
import com.example.checkordering.model.AccountDetails;
import com.example.checkordering.model.AccountRole;
import com.example.checkordering.model.Party;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.springframework.test.util.ReflectionTestUtils;

import java.util.Collections;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

class AccountValidationServiceTest {

    @Mock
    private RestApiGateway restApiGateway;

    @Mock
    private OktaClient oktaClient;

    private AccountValidationService accountValidationService;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        accountValidationService = new AccountValidationService(restApiGateway, oktaClient);

        // Set necessary properties
        ReflectionTestUtils.setField(accountValidationService, "partyServiceBaseUrl", "http://party-service");
        ReflectionTestUtils.setField(accountValidationService, "accountServiceBaseUrl", "http://account-service");
        ReflectionTestUtils.setField(accountValidationService, "partyServiceEcifPath", "/ecif/{ecifId}");
        ReflectionTestUtils.setField(accountValidationService, "partyServiceAccountRolePath", "/account-role/{acctId}");
        ReflectionTestUtils.setField(accountValidationService, "accountDetailsPath", "/account-details/{acctId}");
        ReflectionTestUtils.setField(accountValidationService, "partyClient", "partyClient");
        ReflectionTestUtils.setField(accountValidationService, "accountClient", "accountClient");
        ReflectionTestUtils.setField(accountValidationService, "partyReadScope", "party.read");
        ReflectionTestUtils.setField(accountValidationService, "limitedAcctsB2cReadScope", "limited.accts.b2c.read");
    }

    @Test
    void testValidateAccount() throws Exception {
        // Arrange
        String uuid = "test-uuid";
        String acctId = "test-acct-id";
        String ecifId = "test-ecif-id";
        String token = "test-token";

        Party party = new Party();
        party.setParty(new Party.PartyDetails());
        party.getParty().setPartyDomainId("test-domain-id");

        AccountRole accountRole = new AccountRole();
        accountRole.setAccountId(acctId);
        AccountRole.Role role = new AccountRole.Role();
        role.setAccountRoleType("PRIMARY");
        role.setAccountRoleName("OWNER");
        Party.PartyDetails partyDetails = new Party.PartyDetails();
        partyDetails.setPartyDomainId("test-domain-id");
        role.setParty(partyDetails);
        accountRole.setAccountRole(Collections.singletonList(role));

        when(oktaClient.getToken(anyString(), anyString())).thenReturn(token);
        when(restApiGateway.executeGetRequest(contains("/ecif/"), eq(token), eq(Party.class))).thenReturn(party);
        when(restApiGateway.executeGetRequest(contains("/account-role/"), eq(token), eq(AccountRole.class))).thenReturn(accountRole);

        // Act
        boolean result = accountValidationService.validateAccount(uuid, acctId, ecifId);

        // Assert
        assertTrue(result);
        verify(oktaClient).getToken("partyClient", "party.read");
        verify(restApiGateway, times(2)).executeGetRequest(anyString(), eq(token), any());
    }

    @Test
    void testGetAccountDetails() {
        // Arrange
        String acctId = "test-acct-id";
        String affiliate = "test-affiliate";
        String token = "test-token";
        AccountDetails expectedDetails = new AccountDetails();

        when(oktaClient.getToken(anyString(), anyString())).thenReturn(token);
        when(restApiGateway.executeGetRequest(contains("/account-details/"), eq(token), eq(AccountDetails.class))).thenReturn(expectedDetails);

        // Act
        AccountDetails result = accountValidationService.getAccountDetails(acctId, affiliate);

        // Assert
        assertNotNull(result);
        assertEquals(expectedDetails, result);
        verify(oktaClient).getToken("accountClient", "limited.accts.b2c.read");
        verify(restApiGateway).executeGetRequest(anyString(), eq(token), eq(AccountDetails.class));
    }
}
```
