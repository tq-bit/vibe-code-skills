---
name: implement-ui5-basemodel
description: "Load when the UI5 project does not contain a BaseModel.ts (or .js) in the model/ folder — scaffold it from the bundled source — and when creating, modifying, or reviewing UI5 model classes or any frontend code performing OData v4 CRUD operations against the CAP backend. Enforces the BaseModel inheritance and CRUD delegation rules for the model/ folder. Never scaffold when a BaseModel is already in place or was modified by the user."
---

# UI5 BaseModel

The project uses a generic `BaseModel<T>` (extends `JSONModel`) as the single abstraction for OData v4 CRUD operations. All model classes and all entity CRUD calls MUST go through it.

## Scaffolding

When `model/` does not contain a `BaseModel.ts` (or `BaseModel.js`), create it from the source below. Adapt the `@namespace` comment to the target project's component namespace. Never use this source when a BaseModel is already in place or has been modified by the user.

Updated: 24.09.2026

```ts
import Controller from "sap/ui/core/mvc/Controller";
import Context from "sap/ui/model/odata/v4/Context";
import ODataContextBinding from "sap/ui/model/odata/v4/ODataContextBinding";
import ODataListBinding from "sap/ui/model/odata/v4/ODataListBinding";
import ODataModel from "sap/ui/model/odata/v4/ODataModel";
import JSONModel from "sap/ui/model/json/JSONModel";

/**
 * @namespace com.cpro.ui5.ai.documentextraction.upload.model.BaseModel
 */
export default class BaseModel<T> extends JSONModel {
  private modelName: string;
  private initialData: T;
  protected _controller: Controller | null = null;

  constructor(modelName: string, initialData?: T) {
    super();

    this.modelName = modelName;

    if (initialData) {
      this.initialData = initialData;
    }

    this.setData({
      form: {},
      activeItem: initialData || {},
      activeTree: [],
      collection: []
    });
  }

  public register(handler: Controller): void {
    if (!handler) {
      throw new Error("Controller is undefined");
    }

    this._controller = handler;

    // @ts-ignore
    handler.getOwnerComponent().setModel(this, this.modelName);
  }

  public setActiveItem(data: T): void {
    this.setProperty("/activeItem", data);
  }

  public getActiveItem(): T {
    return this.getProperty("/activeItem");
  }

  public setCollection(data: T[]): void {
    this.setProperty("/collection", data);
  }

  public getCollection(): T[] {
    return this.getProperty("/collection");
  }

  public resetActiveItem(): void {
    this.setProperty("/activeItem", this.initialData);
  }

  public setActiveTree(data: unknown[]): void {
    this.setProperty("/activeTree", data);
  }

  public getActiveTree(): unknown[] {
    return this.getProperty("/activeTree") as unknown[];
  }

  public async requestBoundOdataContext(
    sPath: string,
    bindingParams?: object
  ): Promise<{ context: Context; binding: ODataContextBinding }> {
    if (!this._controller) {
      throw new Error("Controller is not registered for this model");
    }

    const oModel = this._controller.getView()?.getModel() as
      | ODataModel
      | undefined;

    if (!oModel) {
      throw new Error("No OData V4 model is available on the view");
    }

    const oContextBinding = oModel.bindContext(
      sPath,
      undefined,
      bindingParams
    ) as ODataContextBinding;

    oContextBinding.initialize();
    await oContextBinding.requestObject();

    const oContext = oContextBinding.getBoundContext() as Context | undefined;

    if (!oContext) {
      oContextBinding.destroy();
      throw new Error(`Entity not found at path: ${sPath}`);
    }

    return {
      context: oContext,
      binding: oContextBinding
    };
  }

  /**
   * Generic method to create an entity via OData
   * @param entitySetPath The OData entity set path (e.g., "/DocumentMetadata")
   * @param data The data to create
   * @returns The created entity
   */
  public async createEntity<TResult = unknown>(
    entitySetPath: string,
    data: object
  ): Promise<TResult> {
    if (!this._controller) {
      throw new Error("Controller is not registered for this model");
    }

    const oModel = this._controller.getView()?.getModel() as ODataModel;
    const oListBinding = oModel.bindList(entitySetPath) as ODataListBinding;
    const oCreatedContext = oListBinding.create(data);
    await oCreatedContext.created();
    return oCreatedContext.getObject() as TResult;
  }

  /**
   * Generic method to delete an entity via OData
   * @param entitySetPath The OData entity set path (e.g., "/DocumentMetadata")
   * @param id The entity ID value
   * @param idProperty The name of the ID property (default: "ID")
   */
  public async deleteEntity(
    entitySetPath: string,
    id: string,
    idProperty: string = "ID"
  ): Promise<void> {
    const sEncodedId = encodeURIComponent(id);
    const sPath = `${entitySetPath}(${idProperty}='${sEncodedId}')`;

    const { context: oContext, binding: oContextBinding } = await this.requestBoundOdataContext(
      sPath
    );

    try {
      await oContext.delete("$auto");
    } finally {
      oContextBinding.destroy();
    }
  }

  /**
   * Generic method to get a single entity via OData
   * @param entitySetPath The OData entity set path (e.g., "/DocumentMetadata")
   * @param id The entity ID value
   * @param selectFields Array of fields to select (optional)
   * @param idProperty The name of the ID property (default: "ID")
   * @returns The fetched entity
   */
  public async getEntity<TResult = unknown>(
    entitySetPath: string,
    id: string,
    selectFields?: string[],
    idProperty: string = "ID"
  ): Promise<TResult> {
    const sEncodedId = encodeURIComponent(id);
    const sPath = `${entitySetPath}(${idProperty}='${sEncodedId}')`;
    const bindingParams = selectFields?.length
      ? { $select: selectFields }
      : undefined;

    const { context: oContext, binding: oContextBinding } = await this.requestBoundOdataContext(
      sPath,
      bindingParams
    );

    try {
      const entity = oContext.getObject() as TResult;

      if (!entity) {
        throw new Error(`Entity not found at ${entitySetPath} with ${idProperty}=${id}`);
      }

      return entity;
    } finally {
      oContextBinding.destroy();
    }
  }

  /**
   * Generic method to update an entity via OData
   * @param entitySetPath The OData entity set path (e.g., "/DocumentMetadata")
   * @param id The entity ID value
   * @param data Object containing properties to update
   * @param idProperty The name of the ID property (default: "ID")
   * @returns The updated entity
   */
  public async updateEntity<TResult = unknown>(
    entitySetPath: string,
    id: string,
    data: Record<string, unknown>,
    idProperty: string = "ID"
  ): Promise<TResult> {
    const sEncodedId = encodeURIComponent(id);
    const sPath = `${entitySetPath}(${idProperty}='${sEncodedId}')`;

    const { context: oContext, binding: oContextBinding } = await this.requestBoundOdataContext(
      sPath
    );

    try {
      for (const [key, value] of Object.entries(data)) {
        if (value !== undefined) {
          oContext.setProperty(key, value);
        }
      }
      // @ts-ignore
      await oContext.getModel().submitBatch("$auto");
      return oContext.getObject() as TResult;
    } finally {
      oContextBinding.destroy();
    }
  }
}
```

## Enforcement rules

1. Every class in the `model/` folder MUST extend `BaseModel<T>` with a concrete type parameter. A class extending `JSONModel` directly is a violation. The only permitted exception is `BaseModel` itself.
2. All CRUD operations on the OData v4 backend MUST be implemented as methods on a `BaseModel` subclass using the inherited generic operations:
   - Create: `createEntity<TResult>(entitySetPath, data)` — internally bindList → create → `await created()` → getObject().
   - Delete: `deleteEntity(entitySetPath, id, idProperty?)`
   - Read: `getEntity<TResult>(entitySetPath, id, selectFields?, idProperty?)`
   - Update: `updateEntity<TResult>(entitySetPath, id, data, idProperty?)`
   - Context reads: `requestBoundOdataContext(sPath, bindingParams?)`
3. Controllers MUST call these typed model methods. The following patterns in controllers or views are violations and MUST be flagged in review:
   - Direct `oModel.bindList(...)`, `oModel.bindContext(...)`, or manual `ODataListBinding.create(...)`.
   - Direct `Context.delete(...)` or `Context.setProperty(...)` outside BaseModel.
   - Raw `oModel.submitBatch(...)` calls outside BaseModel.
4. New model classes MUST follow the established structure: constructor calls `super(modelName, initialData)`, state is managed only through `setActiveItem`, `getActiveItem`, `setCollection`, `getCollection`, `resetActiveItem`, `setActiveTree`, `getActiveTree`, and the model is attached via `register(controller)` — never via `setModel` calls scattered in controllers.
5. If a required operation is not covered by BaseModel (e.g., a bound action or function import), add a method to the relevant BaseModel subclass that encapsulates the raw OData call and its error handling. NEVER place the raw call in a controller.

## Review checklist

- Does every file in `model/` extend `BaseModel<T>`?
- Does any controller hold a reference to `ODataModel` or `ODataListBinding` beyond a BaseModel subclass?
- Is every CRUD path routed through `createEntity` / `getEntity` / `updateEntity` / `deleteEntity`?
- Are bindings created by `requestBoundOdataContext` destroyed (BaseModel already handles this in finally blocks — verify new code does too)?
- Are `await` and error handling preserved around the async BaseModel calls?

Return all violations to the user.
