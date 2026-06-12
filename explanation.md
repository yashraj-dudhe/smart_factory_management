from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List
import models, schemas
from database import get_db
from auth import get_current_user

router = APIRouter(prefix="/alerts", tags=["Alerts"])

@router.get("/", response_model=List[schemas.Alert])
def read_active_alerts(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    limit = max(1, min(limit, 100))
    alerts = (
        db.query(models.Alert)
        .filter(models.Alert.is_resolved == False)
        .order_by(models.Alert.created_at.desc(), models.Alert.id.desc())
        .offset(skip)
        .limit(limit)
        .all()
    )
    return alerts

@router.get("/{machine_id}", response_model=List[schemas.Alert])
def read_machine_alerts(machine_id: int, active_only: bool = True, db: Session = Depends(get_db)):
    query = db.query(models.Alert).filter(models.Alert.machine_id == machine_id)
    if active_only:
        query = query.filter(models.Alert.is_resolved == False)
    return query.all()

@router.patch("/{id}/resolve", response_model=schemas.Alert)
def resolve_alert(id: int, db: Session = Depends(get_db), current_user: dict = Depends(get_current_user)):
    alert = db.query(models.Alert).filter(models.Alert.id == id).first()
    if not alert:
        raise HTTPException(status_code=404, detail="Alert not found")
    alert.is_resolved = True
    db.commit()
    db.refresh(alert)
    return alert


from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from sqlalchemy import func
import models, schemas
from database import get_db

router = APIRouter(prefix="/analytics", tags=["Analytics"])

@router.get("/machines/{machine_id}", response_model=schemas.MachineAnalytics)
def get_machine_analytics(machine_id: int, db: Session = Depends(get_db)):
    machine = db.query(models.Machine).filter(models.Machine.id == machine_id).first()
    if not machine:
        raise HTTPException(status_code=404, detail="Machine not found")
        
    tel_stats = db.query(
        func.count(models.Telemetry.id).label("total_readings"),
        func.avg(models.Telemetry.temperature).label("avg_temp"),
        func.max(models.Telemetry.temperature).label("max_temp")
    ).filter(models.Telemetry.machine_id == machine_id).first()
    
    alerts = db.query(func.count(models.Alert.id)).filter(models.Alert.machine_id == machine_id).scalar()
    
    return {
        "machine_id": machine_id,
        "total_readings": tel_stats.total_readings or 0,
        "avg_temperature": round(tel_stats.avg_temp or 0.0, 2),
        "max_temperature": round(tel_stats.max_temp or 0.0, 2),
        "total_alerts": alerts or 0
    }

@router.get("/production-lines/{line_id}", response_model=schemas.ProductionLineAnalytics)
def get_line_analytics(line_id: int, db: Session = Depends(get_db)):
    line = db.query(models.ProductionLine).filter(models.ProductionLine.id == line_id).first()
    if not line:
        raise HTTPException(status_code=404, detail="Line not found")
        
    machines = db.query(models.Machine.id).filter(models.Machine.production_line_id == line_id).all()
    machine_ids = [m.id for m in machines]
    
    alert_count = 0
    if machine_ids:
        alert_count = db.query(func.count(models.Alert.id)).filter(models.Alert.machine_id.in_(machine_ids)).scalar()
        
    return {
        "line_id": line_id,
        "total_machines": len(machine_ids),
        "total_alerts": alert_count or 0
    }
from fastapi import APIRouter, HTTPException, Depends
from pydantic import BaseModel
import os

from auth import create_access_token, get_current_user

router = APIRouter(prefix="/auth", tags=["Auth"])


class LoginRequest(BaseModel):
    username: str
    password: str


@router.post("/login")
def login(req: LoginRequest):
    admin_user = os.getenv("ADMIN_USER")
    admin_pass = os.getenv("ADMIN_PASSWORD")
    if not admin_user or not admin_pass:
        raise HTTPException(status_code=500, detail="Admin credentials not configured")
    if req.username != admin_user or req.password != admin_pass:
        raise HTTPException(status_code=401, detail="Invalid username or password")
    token = create_access_token({"sub": req.username})
    return {"access_token": token, "token_type": "bearer"}



from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List
import models, schemas
from database import get_db
from services.health_score import calculate_health_score
from auth import get_current_user

router = APIRouter(prefix="/machines", tags=["Machines"])

@router.post("/", response_model=schemas.Machine)
def create_machine(machine: schemas.MachineCreate, db: Session = Depends(get_db), current_user: dict = Depends(get_current_user)):
    db_mach = models.Machine(**machine.dict())
    db.add(db_mach)
    db.commit()
    db.refresh(db_mach)
    return db_mach

@router.get("/", response_model=List[schemas.Machine])
def read_machines(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    machines = db.query(models.Machine).offset(skip).limit(limit).all()
    return machines

@router.get("/{id}", response_model=schemas.Machine)
def read_machine(id: int, db: Session = Depends(get_db)):
    machine = db.query(models.Machine).filter(models.Machine.id == id).first()
    if machine is None:
        raise HTTPException(status_code=404, detail="Machine not found")
    return machine


@router.delete("/{id}")
def delete_machine(id: int, db: Session = Depends(get_db), current_user: dict = Depends(get_current_user)):
    machine = db.query(models.Machine).filter(models.Machine.id == id).first()
    if machine is None:
        raise HTTPException(status_code=404, detail="Machine not found")

    db.query(models.Telemetry).filter(models.Telemetry.machine_id == id).delete(synchronize_session=False)
    db.query(models.Alert).filter(models.Alert.machine_id == id).delete(synchronize_session=False)
    db.delete(machine)
    db.commit()
    return {"ok": True, "deleted_machine_id": id}

@router.get("/{machine_id}/health", response_model=schemas.HealthScoreResponse)
def read_machine_health(machine_id: int, db: Session = Depends(get_db)):
    machine = db.query(models.Machine).filter(models.Machine.id == machine_id).first()
    if not machine:
        raise HTTPException(status_code=404, detail="Machine not found")
        
    latest_telemetry = db.query(models.Telemetry).filter(models.Telemetry.machine_id == machine_id).order_by(models.Telemetry.recorded_at.desc()).first()
    
    score, status = calculate_health_score(latest_telemetry)
    return {"machine_id": machine_id, "health_score": score, "status": status}

@router.get("/verify")
def verify(current_user: dict = Depends(get_current_user)):
    return {"ok": True, "user": current_user}
my role was backend core give me script to show that i am created 15 endpoints , give me script to say that is short, to the point and concise
