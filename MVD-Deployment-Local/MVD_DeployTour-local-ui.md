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

## 3. Configure the local backend

Open the project and modify the `api.config.local-backend.ts` file to add the `managementBEContextPath` constant, which is required to start the application.

![Backend configuration example](./images/deploy_ui/ui_backend_config.png)

The `basePath` and `PARTICIPANT_ID` constants are configured in the project's `angular.json` file. By default, they appear as shown in the image.

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

Modify the `proxy.local-backend.conf.json` file and update the `target` property to point to the URL of the environment where the backend is located.

![Proxy configuration example](./images/deploy_ui/ui_proxy_config.png)
