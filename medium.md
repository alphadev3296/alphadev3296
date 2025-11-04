# Building a Generic CRUD Base Class for Scalable Python Backends

One common pain point in backend development is writing the same CRUD (Create, Read, Update, Delete) logic again and again for each data model.  
As your project scales, this duplication slows you down and makes maintenance harder.

This article explains how to build and use a **Generic CRUD Base Class** in Python, using **Pydantic** and **PyMongo**.  
You’ll see how it simplifies data access layers and keeps your code consistent across models.

You can view the full working source code in the open repository:  
👉 [GitHub: alphadev3296/fastapi-prototype-healtymeal-copilot-api](https://github.com/alphadev3296/fastapi-prototype-healtymeal-copilot-api)
---

## Motivation: Why a Generic CRUD Layer?

When you start building a backend system, each model gets its own CRUD logic. For example, your `User`, `Order`, and `MealPlan` schemas might each have independent logic to handle:

- Creating a new record  
- Fetching by ID  
- Updating by ID  
- Deleting or counting records  
- Searching with filters  

This results in code duplication and higher maintenance cost.

The solution is to abstract these repeating patterns into a **type-safe, reusable CRUD base** that can easily plug into any new model. Our goal: define CRUD once, use it everywhere.

---

## The Generic CRUD Base

Here’s a breakdown of the core components.

### 1. Type-Parametrized CRUD Definitions

We define the base class using Python’s **type parameters** and **Pydantic models**:

```python
from typing import Any, override

from pydantic import BaseModel
from pymongo import IndexModel
from pymongo.database import Database

from app.models.common.base_models import TimeStampedModel
from app.models.common.object_id import PyObjectId
from app.schemas.errors import ErrorCode404, NotFoundException
from app.utils.datetime import DatetimeUtil


class GenericCRUDBase[DBSchema: BaseModel, ReadSchema: BaseModel, UpdateSchema: BaseModel]:
    """
    Base class for CRUD operations.

    Operations:
        get: Get an object by ID.
        search: Search for objects that match the query.
        count: Count the number of objects that match the query.
        create: Create a new object.
        update: Update an object by ID.
        delete: Delete an object by ID.
        delete_many: Delete all objects that match the query.
        clear: Clear all objects in the collection.
    """

    def __init__(
        self,
        db: Database[dict[str, Any]],
        collection_name: str,
        read_schema: type[ReadSchema],
        indexes: list[list[str]] | None = None,
    ) -> None:
        """
        Initialize the CRUDBase class.

        Args:
            db (Database[dict[str, Any]]): The database to use.
            collection_name (str): The name of the collection to use.
            read_schema (type[ReadSchema]): The schema to use for reading objects.
        """
        self.db = db
        self.coll = db.get_collection(collection_name)
        self.read_schema = read_schema
        self.indexes = indexes

    def create_indexes(self) -> None:
        if self.indexes:
            self.coll.create_indexes(
                [
                    IndexModel(
                        index,
                        unique=True,
                    )
                    for index in self.indexes
                ]
            )

    def get(self, obj_id: PyObjectId | str) -> ReadSchema:
        """
        Get an object by ID.

        Args:
            obj_id (PyObjectId | str): The ID of the object to get.

        Returns:
            ReadSchema: The object.

        Exceptions:
            HTTPException: If the object is not found.
        """
        doc = self.coll.find_one({"_id": PyObjectId(obj_id)})
        if doc is None:
            raise NotFoundException(
                error_code=ErrorCode404.NOT_FOUND,
                message="Item not found",
            )
        return self.read_schema.model_validate(doc)

    def search(self, query: dict[str, Any] | None = None) -> list[ReadSchema]:
        """
        Search for objects that match the query.

        Args:
            query (dict[str, Any]): The query to match objects.

        Returns:
            list[ReadSchema]: A list of objects that match the query.
        """
        cursor = self.coll.find(query or {})
        return [self.read_schema.model_validate(doc) for doc in cursor]

    def count(self, query: dict[str, Any] | None = None) -> int:
        """
        Count the number of objects that match the query.

        Args:
            query (dict[str, Any]): The query to match objects.

        Returns:
            int: The number of objects that match the query.
        """
        return self.coll.count_documents(query or {})

    def create(self, obj_create: DBSchema) -> ReadSchema:
        """
        Create a new object.

        Args:
            obj_create (DBSchema): The object to create.

        Returns:
            ReadSchema: The created object.
        """
        obj_dict = obj_create.model_dump()
        res = self.coll.insert_one(obj_dict)
        obj_dict["_id"] = res.inserted_id
        return self.read_schema.model_validate(obj_dict)

    def update(self, obj_id: PyObjectId | str, obj_update: UpdateSchema) -> ReadSchema:
        """
        Update an object by ID.

        Args:
            obj_update (UpdateSchema): The updated object.
            obj_id (PyObjectId): The ID of the object to update.

        Returns:
            ReadSchema: The updated object.

        Exceptions:
            HTTPException: If the object is not found.
        """
        obj_dict = obj_update.model_dump(exclude_unset=True)
        res = self.coll.update_one({"_id": PyObjectId(obj_id)}, {"$set": obj_dict})

        if res.matched_count == 0:
            raise NotFoundException(
                error_code=ErrorCode404.NOT_FOUND,
                message="Item not found",
            )

        doc = self.coll.find_one({"_id": PyObjectId(obj_id)})
        return self.read_schema.model_validate(doc)

    def delete(self, obj_id: PyObjectId | str) -> None:
        """
        Delete an object by ID.

        Args:
            obj_id (PyObjectId | str): The ID of the object to delete.

        Exceptions:
            HTTPException: If the object is not found.
        """
        res = self.coll.delete_one({"_id": PyObjectId(obj_id)})
        if res.deleted_count == 0:
            raise NotFoundException(
                error_code=ErrorCode404.NOT_FOUND,
                message="Item not found",
            )

    def delete_many(self, query: dict[str, Any]) -> int:
        """
        Delete all objects that match the query.

        Args:
            query (dict[str, Any]): The query to match objects.

        Returns:
            int: The number of deleted objects.
        """
        res = self.coll.delete_many(query)
        return res.deleted_count

    def clear(self) -> None:
        """
        Clear all objects in the collection.
        """
        self.coll.delete_many({})


```

Each CRUD instance is bound to three schemas:
- **DBSchema** – The schema used for insertion (complete record structure).
- **ReadSchema** – The schema used for retrieval (typically includes `_id` and derived fields).
- **UpdateSchema** – The schema defining updatable fields (values can be optional).

By introducing generics, this base class remains fully **type-safe** across all models.

---

### 2. Core CRUD Operations

The CRUD base implements the six fundamental operations:

| Operation | Method | Description |
|------------|---------|-------------|
| Create | `create()` | Inserts a new document into MongoDB |
| Read | `get()` | Fetches a document by ID |
| Update | `update()` | Updates specific fields |
| Delete | `delete()` | Deletes by ID |
| Search | `search()` | Finds documents matching a query |
| Count | `count()` | Counts documents matching a filter |

Each method leverages Pydantic’s `.model_validate()` for schema validation and PyMongo for direct collection operations.

This abstraction completely eliminates repetitive CRUD logic for every collection.

---

## Extending with TimeStamped Support

A second class, `TimeStampedCRUDBase`, extends our generic CRUD with automatic timestamp fields (`created_at` and `updated_at`):

```python
class TimeStampedCRUDBase[
    DBSchema: TimeStampedModel,
    ReadSchema: BaseModel,
    UpdateSchema: BaseModel,
](
    GenericCRUDBase[
        DBSchema,
        ReadSchema,
        UpdateSchema,
    ]
):
    def __init__(
        self,
        db: Database[dict[str, Any]],
        collection_name: str,
        read_schema: type[ReadSchema],
        indexes: list[list[str]] | None = None,
    ) -> None:
        super().__init__(db, collection_name, read_schema, indexes)

    @override
    def create(self, obj_create: DBSchema) -> ReadSchema:
        """
        Create a new object.

        Args:
            obj_create (CreateSchema): The object to create.

        Returns:
            ReadSchema: The created object.
        """
        obj_dict = obj_create.model_dump()

        # Set created_at and updated_at
        obj_dict[TimeStampedModel.Field.created_at] = DatetimeUtil.get_current_timestamp()

        # Insert
        res = self.coll.insert_one(obj_dict)
        obj_dict["_id"] = res.inserted_id
        return self.read_schema.model_validate(obj_dict)

    @override
    def update(self, obj_id: PyObjectId | str, obj_update: UpdateSchema) -> ReadSchema:
        obj_dict = obj_update.model_dump(exclude_unset=True)

        # Set updated_at
        obj_dict[TimeStampedModel.Field.updated_at] = DatetimeUtil.get_current_timestamp()

        # Update
        res = self.coll.update_one({"_id": PyObjectId(obj_id)}, {"$set": obj_dict})
        if res.matched_count == 0:
            raise NotFoundException(
                error_code=ErrorCode404.NOT_FOUND,
                message="Item not found",
            )

        return self.get(obj_id=obj_id)
```

Before insertion or update, it enriches the document with the current timestamp from a utility helper:

```python
obj_dict[TimeStampedModel.Field.created_at] = DatetimeUtil.get_current_timestamp()
```

This pattern is especially useful for tracking record lifecycles without modifying each model manually.

---

## Example: Building CRUD for `MealPlan`

To demonstrate real usage, let’s create a `MealPlanCRUD` that manages meal plans for each day of the week. The schema defines attributes such as plan name, meals, and total calories.

Here’s how the model looks conceptually:

```python
class MealPlan(BaseModel):
    plan_name: str
    description: str
    day: DayOfWeek
    breakfast: Meal | None
    lunch: Meal | None
    dinner: Meal | None
```

To bind it to the CRUD layer:

```python
class MealPlanCRUD(TimeStampedCRUDBase[MealPlan, MealPlanRead, MealPlanUpdate]):
    def __init__(self, db: Database[dict[str, Any]]) -> None:
        super().__init__(
            db=db,
            collection_name="meal_plan",
            read_schema=MealPlanRead,
            indexes=[[MealPlan.Field.plan_name.value]],
        )
```

With this minimal class, you now instantly get:
- Automatic indexes (like on `plan_name`)
- Simple method calls:  
  ```python
  meal_plan_crud.create(meal_plan)
  meal_plan_crud.get(id)
  meal_plan_crud.update(id, payload)
  meal_plan_crud.delete(id)
  ```

No redundant CRUD code—completely DRY and type-safe.

---

## Integration in Business Services

Once the CRUD layer is in place, you can plug it into higher-level services for business logic.  

For example, `MealPlanService` can use the `MealPlanCRUD` for persistence and version logging:

```python
class MealPlanService:
    def __init__(self, db: Database[dict[str, Any]]) -> None:
        self.meal_plan_crud = MealPlanCRUD(db=db)
    
    def create_meal_plan(self, request: CreateMealPlanRequest) -> MealPlanRead:
        return self.meal_plan_crud.create(
            obj_create=request.to_meal_plan(version=Version.initial_version())
        )
```

This design clearly separates **data access** (CRUD layer) from **business logic** (service layer). It’s simple, powerful, and highly maintainable.

---

## Key Benefits of This Approach

1. **Code Reuse** – Define once, reuse for every entity.  
2. **Consistency** – Unified CRUD behavior across all collections.  
3. **Flexibility** – Extend easily with versioning, timestamps, or domain-specific constraints.  
4. **Type Safety** – Thanks to Pydantic + type hints, models are validated automatically.  
5. **Maintainability** – Small changes (like switching DB or adding audit logs) apply system-wide.

---

## When to Use This Pattern

This approach works best for backends that:
- Use **Python + Pydantic + MongoDB (PyMongo)**.  
- Have many data models with standard CRUD behavior.  
- Require clean separation between layers (CRUD, service, and schema).  

If you’re building an API-driven backend with FastAPI or similar frameworks, this structure integrates cleanly.

---

## Conclusion

A **Generic CRUD Base Class** might look like a small architectural optimization, but for long-running backend systems, it significantly improves maintainability and developer productivity.

Once set up, you can add new models by writing only their Pydantic schemas—CRUD operations will come for free.

To explore the full implementation with realistic use cases, timestamps, and version tracking, see the open-source repository:

**GitHub Repository:** [alphadev3296/fastapi-prototype-healtymeal-copilot-api](https://github.com/alphadev3296/fastapi-prototype-healtymeal-copilot-api)
