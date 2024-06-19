#Setup FastAPI and PyMongo
pip install fastapi uvicorn pymongo bcrypt

#Create a main.py file:
from fastapi import FastAPI, HTTPException, Depends
from pymongo import MongoClient
from pydantic import BaseModel
from bson import ObjectId
from typing import List
import bcrypt

app = FastAPI()

# Database connection
client = MongoClient("mongodb://localhost:27017/")
db = client["user_database"]
users_collection = db["users"]
ids_collection = db["ids"]

class User(BaseModel):
    username: str
    email: str
    password: str

class Login(BaseModel):
    email: str
    password: str

class LinkID(BaseModel):
    user_id: str
    linked_id: str

# Utility function to hash passwords
def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt())

# Utility function to verify passwords
def verify_password(plain_password: str, hashed_password: str) -> bool:
    return bcrypt.checkpw(plain_password.encode('utf-8'), hashed_password)

@app.post("/register")
async def register(user: User):
    if users_collection.find_one({"email": user.email}):
        raise HTTPException(status_code=400, detail="Email already registered")
    
    hashed_password = hash_password(user.password)
    user_data = {
        "username": user.username,
        "email": user.email,
        "password": hashed_password
    }
    users_collection.insert_one(user_data)
    return {"message": "User registered successfully"}

@app.post("/login")
async def login(login: Login):
    user = users_collection.find_one({"email": login.email})
    if not user or not verify_password(login.password, user["password"]):
        raise HTTPException(status_code=400, detail="Invalid email or password")
    return {"message": "Login successful"}

@app.post("/link_id")
async def link_id(link_id: LinkID):
    user = users_collection.find_one({"_id": ObjectId(link_id.user_id)})
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    ids_collection.insert_one({"user_id": link_id.user_id, "linked_id": link_id.linked_id})
    return {"message": "ID linked successfully"}

@app.get("/joined_data")
async def get_joined_data():
    pipeline = [
        {
            "$lookup": {
                "from": "ids",
                "localField": "_id",
                "foreignField": "user_id",
                "as": "linked_ids"
            }
        }
    ]
    joined_data = list(users_collection.aggregate(pipeline))
    for data in joined_data:
        data["_id"] = str(data["_id"])
        for id_data in data["linked_ids"]:
            id_data["_id"] = str(id_data["_id"])
    return joined_data

@app.delete("/delete_user/{user_id}")
async def delete_user(user_id: str):
    users_collection.delete_one({"_id": ObjectId(user_id)})
    ids_collection.delete_many({"user_id": user_id})
    return {"message": "User and associated data deleted successfully"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000)


    #Running the Application
     uvicorn main:app --reload

