

# Fiori Application Configuration with Separate CAP Node.js Service on BTP with XSUAA Authentication via AppRouter

**Authors:** Daniel Cabral and Gabriel Bernardo

Most of the available documentation focuses on full-stack projects. This document aims to detail the configuration for projects where the backend and frontend are separated.

## Step by Step

### 1. **Creating the CAP Backend**

-   In the creation wizard, select the options to create with standalone AppRouter (this may be optional, but we did not test a scenario without it), Connectivity Services, Destination Service, and authentication via XSUAA (this last option may not be available or necessary).
    
-   After setting up the project, configure the routes on the xs-security.json, especially the redirect URIs section, containing the URLs that will call the application. In theory, generic BTP URLs with asterisks would suffice, but in the example, it was necessary to specifically add the Fiori application URL.
    
-   It is necessary to create and later assign the UAA user role.
    
-   For better interaction in freestyle apps, install the dependency `@cap-js-community/odata-v2-adapter`.
    
-   In the end, we will have something like:
	 
	 **mta.yaml:**
	 ```yaml
	 _schema-version: 3.3.0
	 ID: test1
	 version: 1.0.0
	 description: "A simple CAP project."
	 parameters:
	   enable-parallel-deployments: true
	 build-parameters:
	   before-all:
	     - builder: custom
	       commands:
	         - npm ci
	         - npx cds build --production
	 modules:
	   - name: test1-srv
	     type: nodejs
	     path: gen/srv
	     parameters:
	       buildpack: nodejs_buildpack
	       readiness-health-check-type: http
	       readiness-health-check-http-endpoint: /health
	     build-parameters:
	       builder: npm
	     provides:
	       - name: srv-api # required by consumers of CAP services (e.g. approuter)
	         properties:
	           srv-url: ${default-url}
	     requires:
	       - name: test1-auth
	       - name: test1-db
	       - name: test1-connectivity
	       - name: test1-destination

	   - name: test1-db-deployer
	     type: hdb
	     path: gen/db
	     parameters:
	       buildpack: nodejs_buildpack
	     requires:
	       - name: test1-db

	   - name: test1
	     type: approuter.nodejs
	     path: app/router
	     parameters:
	       keep-existing-routes: true
	       disk-quota: 256M
	       memory: 256M
	     requires:
	       - name: srv-api
	         group: destinations
	         properties:
	           name: srv-api # must be used in xs-app.json as well
	           url: ~{srv-url}
	           forwardAuthToken: true
	       - name: test1-auth
	       - name: test1-destination
	     provides:
	       - name: app-api
	         properties:
	           app-protocol: ${protocol}
	           app-uri: ${default-uri}

	 resources:
	   - name: test1-auth
	     type: org.cloudfoundry.managed-service
	     parameters:
	       service: xsuaa
	       service-plan: application
	       path: ./xs-security.json
	       config:
	         xsappname: test1-${org}-${space}
	         tenant-mode: dedicated
	   - name: test1-db
	     type: com.sap.xs.hdi-container
	     parameters:
	       service: hana
	       service-plan: hdi-shared
	   - name: test1-connectivity
	     type: org.cloudfoundry.managed-service
	     parameters:
	       service: connectivity
	       service-plan: lite
	   - name: test1-destination
	     type: org.cloudfoundry.managed-service
	     parameters:
	       service: destination
	       service-plan: lite
	 ```

	 **xs-secutity.json:** 
	```json
	{
	  "tenant-mode": "dedicated",
	  "scopes": [
	    {
	      "name": "uaa.user",
	      "description": "UAA"
	    }
	  ],
	  "attributes": [
	    {
	      "name": "firstname",
	      "description": "User First Name",
	      "valueType": "string",
	      "valueRequired": false
	    },
	    {
	      "name": "lastname",
	      "description": "User Last Name",
	      "valueType": "string",
	      "valueRequired": false
	    },
	    {
	      "name": "Groups",
	      "description": "User groups",
	      "valueType": "string",
	      "valueRequired": false
	    },
	    {
	      "name": "Roles",
	      "description": "User roles",
	      "valueType": "string",
	      "valueRequired": false
	    }
	  ],
	  "role-templates": [
	    {
	      "name": "Token_Exchange",
	      "description": "UAA",
	      "scope-references": [
	        "uaa.user"
	      ]
	    }
	  ],
	  "oauth2-configuration": {
	    "redirect-uris": [
	      "MY_SPECIFIC_URLS",
	      "https://*.hana.ondemand.com/**",
	      "https://*.applicationstudio.cloud.sap/**",
	      "http://localhost:5000/**",
	      "https://*.ondemand.com/**",
	      "http://*.localhost/**"
	    ]
	  }
	}
	```
	
	**package.json:** 
		
	```json
		{
		  "name": "test1",
		  "version": "1.0.0",
		  "description": "A simple CAP project.",
		  "repository": "<Add your repository here>",
		  "license": "UNLICENSED",
		  "private": true,
		  "dependencies": {
		    "@cap-js-community/odata-v2-adapter": "^1.14.1",
		    "@cap-js/hana": "^1",
		    "@sap/cds": "^8",
		    "@sap/xssec": "^4",
		    "express": "^4",
		    "rimraf": "^6.0.1"
		  },
		  "devDependencies": {
		    "@cap-js/cds-types": "^0.8.0",
		    "@cap-js/sqlite": "^1",
		    "@sap/cds-dk": "^8"
		  },
		  "scripts": {
		    "start": "cds-serve",
		    "build": "rimraf resources mta_archives && mbt build --mtar archive",
		    "deploy": "cf deploy mta_archives/archive.mtar --f --retries 1"
		  },
		  "cds": {
		    "requires": {
		      "auth": {
		        "kind": "xsuaa"
		      },
		      "connectivity": true,
		      "destinations": true
		    },
		    "cov2ap": {
		      "plugin": true,
		      "bodyParserLimit": "300mb"
		    },
		    "sql": {
		      "native_hana_associations": false
		    }
		  }
		}
	```
 
### 2. **Creating the Destination in BTP**
The Destination should be created from the CAP service URL (not the AppRouter URL), using the following parameters:
```
	#
	#Wed Feb 26 02:07:25 UTC 2025
	Type=HTTP
	HTML5.DynamicDestination=true
	Description=TESTE_CAP
	Authentication=NoAuthentication
	HTML5.ForwardAuthToken=true
	WebIDEEnabled=true
	ProxyType=Internet
	URL=https\://MY_URL.COM
	Name=TESTE_CAP
	WebIDEUsage=odata_gen
```

### 3. **Creating the Fiori App**
After creating the Destination, proceed with the Fiori App creation via the wizard. Select the "Basic" option for freestyle applications. In Data Source, select "No". When prompted to create an AppRouter, enable it, then establish a connection with the created Destination.

To enable the Fiori application to authenticate with the CAP backend, you must bind it to the same XSUAA instance used by the backend. This can be done as shown in the example below (pay attention to 'testebackend-auth' and to 'org.cloudfoundry.existing-service'):

**mta.yaml:**
```yaml
_schema-version: "3.2"
ID: testefrontend
description: Generated by Fiori Tools
version: 0.0.1
modules:
- name: testefrontend-destination-content
  type: com.sap.application.content
  requires:
  - name: testefrontend-destination-service
    parameters:
      content-target: true
  - name: testefrontend-repo-host
    parameters:
      service-key:
        name: testefrontend-repo-host-key
  - name: testefrontend-uaa
    parameters:
      service-key:
        name: testefrontend-app-content-testebackend-auth-credentials
  parameters:
    content:
      instance:
        destinations:
        - Name: testefrontend_html_repo_host
          ServiceInstanceName: testefrontend-html5-service
          ServiceKeyName: testefrontend-repo-host-key
          sap.cloud.service: testefrontend
        - Authentication: OAuth2UserTokenExchange
          Name: testefrontend_uaa
          ServiceInstanceName: testebackend-auth
          ServiceKeyName: testefrontend-app-content-testebackend-auth-credentials
          sap.cloud.service: testefrontend
        existing_destinations_policy: update
  build-parameters:
    no-source: true
- name: testefrontend-app-content
  type: com.sap.application.content
  path: .
  requires:
  - name: testefrontend-repo-host
    parameters:
      content-target: true
  build-parameters:
    build-result: resources
    requires:
    - artifacts:
      - testefrontend.zip
      name: testefrontend
      target-path: resources/
- name: testefrontend
  type: html5
  path: .
  build-parameters:
    build-result: dist
    builder: custom
    commands:
    - npm install
    - npm run build:cf
    supported-platforms: []
resources:
- name: testefrontend-destination-service
  type: org.cloudfoundry.managed-service
  parameters:
    config:
      HTML5Runtime_enabled: true
      init_data:
        instance:
          destinations:
          - Authentication: NoAuthentication
            Name: ui5
            ProxyType: Internet
            Type: HTTP
            URL: https://ui5.sap.com
          existing_destinations_policy: update
      version: 1.0.0
    service: destination
    service-name: testefrontend-destination-service
    service-plan: lite
- name: testefrontend-uaa
  type: org.cloudfoundry.existing-service
  parameters:
    service: xsuaa
    service-name: testebackend-auth
- name: testefrontend-repo-host
  type: org.cloudfoundry.managed-service
  parameters:       
    service: html5-apps-repo
    service-name: testefrontend-html5-service
    service-plan: app-host
parameters:
  deploy_mode: html5-repo
  enable-parallel-deployments: true                                   
```

Configure the `xs-security.json`, `xs-app.json`, and `manifest.json` for connection establishment with route forwarding and token passing.

**xs-secutity.json:**
```json
{
  "xsappname": "project2",
  "tenant-mode": "dedicated",
  "description": "Security profile of called application",
  "scopes": [
    {
      "name": "uaa.user",
      "description": "UAA"
    }
  ],
  "role-templates": [
    {
      "name": "Token_Exchange",
      "description": "UAA",
      "scope-references": [
        "uaa.user"
      ]
    }
  ]
}
```
**xs-app.json:**
```json
{
  "welcomeFile": "/index.html",
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "/TESTE_CAP/(.*)$",
      "target": "/$1",
      "destination": "TESTE_CAP",
      "authenticationType": "xsuaa",
      "csrfProtection": false
    },
    {
      "source": "^/resources/(.*)$",
      "target": "/resources/$1",
      "authenticationType": "none",
      "destination": "ui5"
    },
    {
      "source": "^/test-resources/(.*)$",
      "target": "/test-resources/$1",
      "authenticationType": "none",
      "destination": "ui5"
    },
    {
      "source": "^(.*)$",
      "target": "$1",
      "service": "html5-apps-repo-rt",
      "authenticationType": "xsuaa"
    }
  ]
}
```
**manifest.json:**
```json
{
  "_version": "1.65.0",
  "sap.app": {
    "id": "project2",
    "type": "application",
    "i18n": "i18n/i18n.properties",
    "applicationVersion": {
      "version": "0.0.1"
    },
    "title": "{{appTitle}}",
    "description": "{{appDescription}}",
    "resources": "resources.json",
    "sourceTemplate": {
      "id": "@sap/generator-fiori:basic",
      "version": "1.16.4",
      "toolsId": "b03e269d-47c0-4696-8ca5-b05b9b40254c"
    },
    "dataSources": {
      "TESTE_CAP": {
        "uri": "/TESTE_CAP/odata/v2/catalog/",
        "type": "OData",
        "settings": {
          "localUri": "localService/TESTE_CAP/metadata.xml"
        }
      }
    },
    "crossNavigation": {
      "inbounds": {
        "teste2-teste2": {
          "semanticObject": "teste2",
          "action": "teste2",
          "title": "{{teste2-teste2.flpTitle}}",
          "subTitle": "{{teste2-teste2.flpSubtitle}}",
          "signature": {
            "parameters": {},
            "additionalParameters": "allowed"
          }
        }
      }
    }
  },
  "sap.ui": {
    "technology": "UI5",
    "icons": {
      "icon": "",
      "favIcon": "",
      "phone": "",
      "phone@2": "",
      "tablet": "",
      "tablet@2": ""
    },
    "deviceTypes": {
      "desktop": true,
      "tablet": true,
      "phone": true
    }
  },
  "sap.ui5": {
    "flexEnabled": true,
    "dependencies": {
      "minUI5Version": "1.133.0",
      "libs": {
        "sap.m": {},
        "sap.ui.core": {}
      }
    },
    "contentDensities": {
      "compact": true,
      "cozy": true
    },
    "models": {
      "i18n": {
        "type": "sap.ui.model.resource.ResourceModel",
        "settings": {
          "bundleName": "project2.i18n.i18n"
        }
      },
      "oHanaModel": {
        "type": "sap.ui.model.odata.v2.ODataModel",
        "settings": {
          "defaultOperationMode": "Server",
          "defaultBindingMode": "TwoWay",
          "defaultCountMode": "Request",
          "useBatch": false,
          "disableHeadRequestForToken": true
        },
        "dataSource": "TESTE_CAP",
        "preload": true
      }
    },
    "resources": {
      "css": [
        {
          "uri": "css/style.css"
        }
      ]
    },
    "routing": {
      "config": {
        "routerClass": "sap.m.routing.Router",
        "controlAggregation": "pages",
        "controlId": "app",
        "transition": "slide",
        "type": "View",
        "viewType": "XML",
        "path": "project2.view",
        "async": true,
        "viewPath": "project2.view"
      },
      "routes": [
        {
          "name": "RouteView1",
          "pattern": ":?query:",
          "target": [
            "TargetView1"
          ]
        }
      ],
      "targets": {
        "TargetView1": {
          "id": "View1",
          "name": "View1"
        }
      }
    },
    "rootView": {
      "viewName": "project2.view.App",
      "type": "XML",
      "id": "App"
    }
  },
  "sap.cloud": {
    "public": true,
    "service": "project2"
  }
}
```

## Troubleshooting

-   If encountering a "query does not exist for route..." error, check the backend redirect URIs.
    
-   If a 404 error occurs in Fiori, verify the `xs-app.json` and `manifest.json` configurations.
    
-   If a 401 error occurs when calling Fiori, ensure the Destination is correctly configured.
    
-   If in doubt, check if the necessary roles, e.g., UAA User, are correctly assigned.

- If you're receiving an unexpected response from the metadata request in the Fiori application, the issue is likely due to incorrect XSUAA bindings. This probably means the Fiori application is not bound to the same XSUAA instance as the CAP backend.
- If you're trying to add this method of authentication to an existing project, it is necessary to completely delete everything related to the Fiori application (HTML5 repo, destination service, service keys etc) from Cloud Foundry and then redeploy it with the new configuration. It is not necessary to delete the CAP backend.   
-   If SAP error screens appear when trying to log in via the CAP AppRouter URL, ensure redirect URIs are correctly configured in `xs-security.json`
	```
	Proposed Solution:
	Dear CLIENT,

	Good day to you, thanks for reaching SAP Product Support.

	I've gone through the attached error screenshot.

	The issue is because the redirect domain is missing from redirect-uris parameter of the application's xs-security.json file.

	======

	To fix the issue, please follow KBA 3411842 - "The redirect_uri has an invalid domain" when opening an application in BTP Cloud Foundry - SAP for Me to fix the issue.
	Even though the error message is different, the solution can still be applied.

	You could add the parameter in xs-security.json file of CAP project, and then build the app and redeploy it.
	-----
	for example:

	{
	  "scopes": [],
	  "attributes": [],
	  "role-templates": [],
	  "oauth2-configuration": {

	    "redirect-uris": ["https://<host>.hana.ondemand.com/**"]
	}
	}
	-----
	This will fix the issue.

	[See Also] - https://help.sap.com/docs/btp/sap-business-technology-platform/security-considerations-for-sap-authorization-and-trust-management-service#listing-allowed-redirect-uris
	Kindly let me know if issue remains.

	Best Regards,
	RESPONDER
	```
