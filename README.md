```
package com.example.checkordering.service;

import com.example.checkordering.client.OktaClient;
import com.example.checkordering.model.AccountDetails;
import com.example.checkordering.model.AccountRole;
import com.example.checkordering.model.Party;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.springframework.http.HttpHeaders;
import org.springframework.test.util.ReflectionTestUtils;
import org.springframework.web.client.RestClient;

import java.util.Collections;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

class AccountValidationServiceTest {

    @Mock
    private RestClient restClient;

    @Mock
    private OktaClient oktaClient;

    private AccountValidationService accountValidationService;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        accountValidationService = new AccountValidationService(restClient, oktaClient);

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
        when(restClient.get(anyString(), eq(HttpHeaders.AUTHORIZATION), anyString(), eq(Party.class))).thenReturn(party);
        when(restClient.get(anyString(), eq(HttpHeaders.AUTHORIZATION), anyString(), eq(AccountRole.class))).thenReturn(accountRole);

        // Act
        boolean result = accountValidationService.validateAccount(uuid, acctId, ecifId);

        // Assert
        assertTrue(result);
        verify(oktaClient).getToken("partyClient", "party.read");
        verify(restClient, times(2)).get(anyString(), eq(HttpHeaders.AUTHORIZATION), anyString(), any());
    }

    @Test
    void testGetAccountDetails() {
        // Arrange
        String acctId = "test-acct-id";
        String affiliate = "test-affiliate";
        String token = "test-token";
        AccountDetails expectedDetails = new AccountDetails();

        when(oktaClient.getToken(anyString(), anyString())).thenReturn(token);
        when(restClient.get(anyString(), eq(HttpHeaders.AUTHORIZATION), anyString(), eq(AccountDetails.class))).thenReturn(expectedDetails);

        // Act
        AccountDetails result = accountValidationService.getAccountDetails(acctId, affiliate);

        // Assert
        assertNotNull(result);
        assertEquals(expectedDetails, result);
        verify(oktaClient).getToken("accountClient", "limited.accts.b2c.read");
        verify(restClient).get(anyString(), eq(HttpHeaders.AUTHORIZATION), anyString(), eq(AccountDetails.class));
    }
}
```
