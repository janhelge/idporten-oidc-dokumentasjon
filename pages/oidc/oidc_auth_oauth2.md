---
title: "Authentiseringsnær autorisasjon  med OAuth2 og OpenID Connect"
description: "Authentiseringsnær autorisasjon  med OAuth2 og OpenID Connect"
summary: "I forbindelse med en OpenID Connect-autentisering kan ID-portens OpenID Connect provider også autorisere en tjeneste til å opptre på vegne av innbyggeren opp mot et API tilbudt av en 3.dje-part."
permalink: oidc_auth_oauth2.html

layout: page
sidebar: oidc
---


## Overordna beskrivelse av scenariet

Dette er den klassise Oauth2-flyten, der innbyggeren samtykker - enten eksplisitt eller implisitt - til at tjenesten kan bruke et API på vegne av seg selv.  I ID-porten-sammenheng vil vanligvis samtykket være implisitt, siden det er autentiseringshandlingen som i seg selv tolkes som det informerte samtykket ("Ved å logge inn i tjenesten godtar du at vi henter opplysninger om deg fra NAV"). Vi bruker derfor begrepet *autentiseringsnær autorisasjon*.

For eksplisitte samtykker som skal vare "lenge" ("jeg samtykker til at Banken min kan hente inntektsopplysninger hos Skatteetaten de neste 3 årene") henviser vi til bruk av Samtykkeløsningen i Altinn.


Samtykket, eller autorisasjonen, blir av ID-porten utlevert som et _access_token_. Tjenesten bruker så dette access_tokenet når den skal aksessere APIet.

Hvilket API/ressurs som skal aksesserers, er styrt av _scopes_.  Klienten må vite hvilke(t) scope som hører til den aktuelle API-operasjonen, og må forespørre dette scopet i autorisasjonsforespørselen.

<div class="mermaid">
graph LR
  subgraph 3djepart
    API
  end
  subgraph Difi
    OIDC[OIDC Provider]
  end
  subgraph Kunde
     ny[Tjeneste]
  end
  OIDC -->|3.utsteder token|ny
  Sluttbruker ---|2.autentiserer og autoriserer|OIDC
  ny -->|1. forspør tilgang|OIDC
  ny -->|4.bruker token mot|API
</div>

Følgende aktører inngår:

 Aktør | Beskrivelse | Begrep OIDC | Begrep Oauth2 | Begrep SAML2
 -|-|-|-|-|
 Sluttbruker | Ønsker å logge inn til en offentlig tjeneste | End User | User | End User
 Nett-tjeneste | Sluttbruker-tjeneste tilbudt av en offentlig etat | Relying Party (RP) | Client | Service Provider (SP) |
 ID-porten | ID-porten sin OpenID Connect provider som usteder *access_token* til aktuelle tjenesten| OpenID Provider (OP) | Authorization server (AS) | Identity Provider (IDP)
 | API | 3.part, som tilbyr et API som sluttbrukertjenesten ønsker å benytte | - | Resource server (RS) | - |


 ## Beskrivelse av Oauth2-flyten

<div class="mermaid">
sequenceDiagram
  Sluttbruker ->> Klient: Klikker login-knapp
  Klient ->> Sluttbruker: Redirect med autentiseringsforespørsel
  Sluttbruker ->> OpenID Provider: følg redirect...
  note over Sluttbruker,OpenID Provider: Sluttbruker autentiserer seg (og evt. samtykker til førespurte scopes)
  OpenID Provider ->> Sluttbruker: Redirect med autorisasjonscode
  Sluttbruker ->> Klient: følg redirect...
  Klient ->> OpenID Provider: forespørre token (/token)
  OpenID Provider ->> Klient: id_token + access_token (evt. refresh_token)
  note over Sluttbruker,Klient: Innlogget i tjenesten
  Klient ->> API: bruke API med access_token
  API ->> OpenID Provider: validere token
  OpenID Provider ->> API: token informasjon
  API->>Klient: Resultat av API-operasjon
</div>


Starten av flyten er identisk med [autorisasjonskode-flyten for autentisering](oidc_auth_codeflow.html).
* I **autentiseringsresponsen** fra OpenID Provider får klient også utlevert et *access_token* (og eventuelt et *refresh_token*) som gir tilgang til forespurte scopes.  
* Etter innlogging kan da klienten bruke access_tokenet opp mot det relevante APIet.  
  * Access_token har vanligvis kort levetid (30 sekunder). Dersom tokenet er utløpt, kan klienten forespørre nytt acess_token ved å bruke *refresh_tokenet* mot token-endepunktet til OpenID Provideren.  Det gjennomføres da en klient-autentisering, for å sikre at tokens ikke blir utlevert til feil part.

Forskjellen på *autentisering* (OpenIDConnect) og *autorisasjon* med "plain" Oauth2 er altså minimal:
1. For å sikre at autentisering-oppførselen blir ihht. OpenID Connect-spesifikasjonen **må** man benytte 'openid'-scopet
2. OpenID Connect forholder seg ikke til ressurs-servere /API-er, men man kan fint forespørre ekstra scopes i en OIDC autentiseringsforespørsel, og således oppnå kombinert autorisasjon og autentisering.



## Typer access_token og validering av disse

ID-porten kan utstede to typer access_token.  Ressursservere som mottar access_token som del av en API-forespørsel må validere disse før API-operasjonen kan utføres.

|Type token|Beskrivelse|
|-|-|
|by reference| Tokenet inneholder kun en referanse til autorisasjonen internt i ID-porten.  Ressursserveren må sjekke opp mot tokeninfo-endpunktet til ID-porten for å få om tokenet fremdeles er gyldig, hvem brukeren er, hvilke scopes som ble forespurt, og hvilken klient (inkludert dennes org.nr.) det var utstedt til. |
|by value | Tokenet er såkalt self-contained, dvs. det inneholder all informasjon som man ellers kunne sjekke mot tokeninfo-endpunktet.  Slike tokens kan ikke trekkes tilbake, og bør derfor ha kort levetid |

###  Eksempel på bruk av tokeninfo-endepunktet



Følgende header-parametere må brukes på request:

| Parameter  | Verdi |
| --- | --- |
|Http-metode:|POST|
|Content-type:|application/x-www-form-urlencoded|

Følgende attributter må sendes inn i requesten:

| Attributt  | Verdi |
| --- | --- |
|token|\<Tokenet som skal valideres\>|

Eksempel på request:

```
POST /tokeninfo
Content-type: application/x-www-form-urlencoded

token=fK0dhs5vQsuAUguLL2wxbXEQSE91XbOAL3foY5VR0Uk=
```

Eksempel på en respons ved suksessfull validering av token:

```
{
    "active": true,
    "token_type": "Bearer",
    "expires_in": 556,
    "exp": 1477990301,
    "iat": 1477989701,
    "scope": "global/kontaktinformasjon.read",
    "client_id": "test_rp",
    "client_orgno": "991825827"
}
```

Eksempel på en respons ved feilet validering av token:

```
{
    "active": false
}  
```