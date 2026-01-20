# Local Deployment of MVDS-UI

This document describes the necessary steps to run the **MVDS** project user interface locally.

---

## 1. Clone the repository

Download the interface from the repository:

```bash
git clone https://github.com/AmadeusITGroup/Dataspace_UI.git
```

---

## 2. Install dependencies

From the root of the project, run:

```bash
npm install
```

---

## 3. Configure backend integration

Open the project and edit the `api.config.local-backend.ts` file. Make sure to define the `managementBEContextPath` constant in this file, as it specifies the backend context path required for the application to initialize and communicate properly with backend services.

![Backend configuration example](./images/deploy_ui/ui_backend_config.png)

The `basePath` and `PARTICIPANT_ID` constants are defined in the project's `angular.json` file. These values are set by default as shown in the example image, and are used by the application to connect with the backend.

![Routes configuration example](./images/deploy_ui/ui_backend_constants_config.png)

---

## 4. Start the application

Run one of the following commands:

```bash
npm run start:local-backend-provider
npm run start:local-backend-consumer
```

---

## 5. Configure the proxy to the backend

To ensure that the user interface communicates correctly with the backend, you need to configure the proxy settings. Open the `proxy.local-backend.conf.json` file and locate the `target` property. Set this property to the URL of the backend you want to use. This allows the frontend to forward API requests to the correct backend server.

![Proxy configuration example](./images/deploy_ui/ui_proxy_config.png)

> **Note:** The URL shown in the image above is just a placeholder. Make sure to replace it with the actual backend URL you intend to use.
