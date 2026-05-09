# PulseLink AI/ML System

**Intelligent Location Coordination & Abuse Prevention**

---

## 1. Smart Meetup Prediction

### 1.1 Movement Pattern Analysis

```python
import numpy as np
from sklearn.ensemble import GradientBoostingRegressor
from datetime import datetime, timedelta

class MeetupPredictionService:
    def __init__(self):
        self.model = GradientBoostingRegressor(
            n_estimators=100,
            learning_rate=0.1,
            max_depth=5,
            random_state=42
        )
    
    async def predict_meetup_location(
        self,
        user_ids: List[str],
        meeting_preferences: dict
    ) -> dict:
        """Predict optimal meetup location for group"""
        
        # Fetch recent location history
        histories = await asyncio.gather(*[
            self.get_location_history(uid, days=7)
            for uid in user_ids
        ])
        
        # Extract features
        features = await self.extract_features(user_ids, histories)
        
        # Predict meeting location
        prediction = self.model.predict([features])[0]
        
        # Add safety & crowd analysis
        location = {
            'latitude': prediction[0],
            'longitude': prediction[1],
            'confidence': prediction[2],
            'estimated_arrival_minutes': int(prediction[3]),
            'safety_score': await self.calculate_safety_score(prediction),
            'crowd_level': await self.estimate_crowd_level(prediction),
            'venue_type': await self.suggest_venues(prediction),
        }
        
        return location
    
    async def extract_features(
        self,
        user_ids: List[str],
        histories: List[List[dict]]
    ) -> np.ndarray:
        """Extract ML features from location history"""
        
        features = []
        
        for user_id, history in zip(user_ids, histories):
            # Time-based features
            hours_of_day = [h['timestamp'].hour for h in history]
            day_of_week = [h['timestamp'].weekday() for h in history]
            
            # Spatial features (cluster centers)
            locations = np.array([
                [h['latitude'], h['longitude']] for h in history
            ])
            
            # Calculate home location (most frequent during night)
            night_hours = [h for h in history if h['timestamp'].hour in [22, 23, 0, 1, 2]]
            if night_hours:
                home_lat = np.mean([h['latitude'] for h in night_hours])
                home_lng = np.mean([h['longitude'] for h in night_hours])
            else:
                home_lat, home_lng = locations[0]
            
            # Calculate work location (most frequent during day)
            work_hours = [h for h in history if h['timestamp'].hour in [9, 10, 11, 12, 13, 14, 15, 16, 17]]
            if work_hours:
                work_lat = np.mean([h['latitude'] for h in work_hours])
                work_lng = np.mean([h['longitude'] for h in work_hours])
            else:
                work_lat, work_lng = locations[-1]
            
            # Movement speed
            distances = []
            for i in range(len(history) - 1):
                from geopy.distance import geodesic
                d = geodesic(
                    (history[i]['latitude'], history[i]['longitude']),
                    (history[i+1]['latitude'], history[i+1]['longitude'])
                ).meters
                distances.append(d)
            
            avg_speed = np.mean(distances) if distances else 0
            
            features.append([
                home_lat, home_lng,
                work_lat, work_lng,
                avg_speed,
                np.std(hours_of_day) if hours_of_day else 0,
                len(history),  # Activity density
            ])
        
        return np.array(features).flatten()
    
    async def calculate_safety_score(
        self,
        location: tuple
    ) -> float:
        """Calculate safety score (0-1) for location"""
        lat, lng = location[:2]
        
        # Query crime statistics
        crime_data = await self.get_crime_data(lat, lng)
        
        # Query police stations, hospitals nearby
        amenities = await self.get_nearby_amenities(lat, lng)
        
        # Calculate score
        score = 1.0
        score -= (crime_data['incidents_per_km2'] * 0.1)
        score += (len(amenities['police_stations']) * 0.05)
        score += (len(amenities['hospitals']) * 0.03)
        
        return max(0.0, min(1.0, score))
```

---

## 2. Anomaly Detection (Anti-Stalking)

### 2.1 Behavioral Anomaly Detection

```python
from sklearn.ensemble import IsolationForest
from scipy.stats import zscore

class AnomalyDetectionService:
    def __init__(self):
        self.isolation_forest = IsolationForest(
            contamination=0.05,
            random_state=42
        )
        self.suspicious_patterns = []
    
    async def detect_stalking_behavior(
        self,
        requester_id: str,
        target_id: str
    ) -> float:
        """Detect stalking behavior (0-1 risk score)"""
        
        # Get request history
        requests = await self.get_request_history(
            requester_id, target_id, days=30
        )
        
        if len(requests) < 3:
            return 0.0  # Not enough data
        
        score = 0.0
        
        # Feature 1: Request frequency
        requests_per_day = len(requests) / 30
        if requests_per_day > 10:
            score += 0.4
        elif requests_per_day > 5:
            score += 0.2
        
        # Feature 2: Denial ratio
        denials = sum(1 for r in requests if r.status == 'denied')
        denial_ratio = denials / len(requests)
        if denial_ratio > 0.7:
            score += 0.4  # High rejection = stalking indicator
        
        # Feature 3: Time pattern analysis
        request_times = [r.created_at.hour for r in requests]
        time_variance = np.var(request_times)
        if time_variance < 5:  # Very narrow time window
            score += 0.2
        
        # Feature 4: Request after denial
        recent_denials = [r for r in requests if r.status == 'denied' and 
                         datetime.utcnow() - r.created_at < timedelta(days=1)]
        if recent_denials:
            score += 0.3
        
        return min(score, 1.0)
    
    async def detect_location_spoofing(
        self,
        user_id: str
    ) -> bool:
        """Detect impossible travel (teleportation)"""
        
        # Get last two location updates
        locations = await self.get_recent_locations(user_id, limit=2)
        
        if len(locations) < 2:
            return False
        
        loc1, loc2 = locations[1], locations[0]
        time_diff = (loc2['timestamp'] - loc1['timestamp']).total_seconds()
        
        if time_diff < 1:
            return False  # Same timestamp
        
        # Calculate distance
        from geopy.distance import geodesic
        distance = geodesic(
            (loc1['latitude'], loc1['longitude']),
            (loc2['latitude'], loc2['longitude'])
        ).meters
        
        # Max human speed: 150 m/s (car on highway)
        max_distance = time_diff * 150
        
        if distance > max_distance:
            return True  # Impossible travel detected
        
        return False
    
    async def detect_multi_device_abuse(
        self,
        user_id: str
    ) -> bool:
        """Detect suspicious multi-device activity"""
        
        sessions = await self.get_user_sessions(user_id, hours=24)
        
        # Get locations from each device
        device_locations = {}
        for session in sessions:
            device_locations[session['device_id']] = []
        
        # Check if same user is in multiple locations simultaneously
        for i, session1 in enumerate(sessions):
            for session2 in sessions[i+1:]:
                if session1['device_id'] != session2['device_id']:
                    # Same user, different devices
                    # Get locations at same time
                    loc1 = await self.get_location_at_time(
                        user_id, session1['device_id'], datetime.utcnow()
                    )
                    loc2 = await self.get_location_at_time(
                        user_id, session2['device_id'], datetime.utcnow()
                    )
                    
                    if loc1 and loc2:
                        distance = geodesic(
                            (loc1['latitude'], loc1['longitude']),
                            (loc2['latitude'], loc2['longitude'])
                        ).meters
                        
                        if distance > 1000:  # >1km apart
                            return True
        
        return False
```

---

## 3. Abuse Scoring System

### 3.1 Multi-Factor Risk Scoring

```python
class AbuseScoreService:
    SCORE_WEIGHTS = {
        'stalking': 0.3,
        'location_spoofing': 0.25,
        'rate_limit_abuse': 0.2,
        'multi_device_abuse': 0.15,
        'consent_violation': 0.1,
    }
    
    async def calculate_abuse_score(
        self,
        user_id: str
    ) -> dict:
        """Calculate comprehensive abuse score"""
        
        scores = {
            'stalking': await self.score_stalking_behavior(user_id),
            'location_spoofing': float(await self.detect_location_spoofing(user_id)),
            'rate_limit_abuse': await self.score_rate_limit_abuse(user_id),
            'multi_device_abuse': float(await self.detect_multi_device_abuse(user_id)),
            'consent_violation': await self.score_consent_violations(user_id),
        }
        
        # Calculate weighted final score
        final_score = sum(
            scores[key] * self.SCORE_WEIGHTS[key]
            for key in scores
        )
        
        # Determine action
        action = self.determine_action(final_score)
        
        return {
            'user_id': user_id,
            'overall_score': final_score,
            'component_scores': scores,
            'action': action,
            'severity': self.categorize_severity(final_score),
            'timestamp': datetime.utcnow(),
        }
    
    def determine_action(self, score: float) -> str:
        """Determine action based on score"""
        if score >= 0.9:
            return 'ban'  # Immediate ban
        elif score >= 0.7:
            return 'block'  # Restrict features
        elif score >= 0.5:
            return 'rate_limit'  # Heavy rate limiting
        elif score >= 0.3:
            return 'warn'  # Send warning
        else:
            return 'monitor'  # Continue monitoring
    
    async def score_rate_limit_abuse(
        self,
        user_id: str
    ) -> float:
        """Score rate limit violations"""
        
        violations = await self.get_rate_limit_violations(user_id, days=7)
        
        if not violations:
            return 0.0
        
        # More violations = higher score
        score = min(len(violations) / 10, 1.0)
        
        return score
    
    async def score_consent_violations(
        self,
        user_id: str
    ) -> float:
        """Score consent violations"""
        
        violations = await self.get_consent_violations(user_id, days=7)
        
        if not violations:
            return 0.0
        
        # Accessing denied sessions = violation
        score = min(len(violations) / 5, 1.0)
        
        return score
```

---

## 4. Model Training & Deployment

### 4.1 Training Pipeline

```python
class MLTrainingPipeline:
    async def train_meetup_prediction_model(self):
        """Train meetup prediction model"""
        
        # Collect training data
        training_data = await self.collect_training_data(
            start_date=datetime.utcnow() - timedelta(days=90),
            end_date=datetime.utcnow()
        )
        
        # Extract features
        X, y = await self.extract_training_features(training_data)
        
        # Train model
        model = GradientBoostingRegressor(
            n_estimators=200,
            learning_rate=0.05,
            max_depth=7,
            subsample=0.8
        )
        model.fit(X, y)
        
        # Evaluate
        score = model.score(X, y)
        logger.info(f"Model R² score: {score}")
        
        # Save model
        import joblib
        joblib.dump(model, 'models/meetup_prediction_v1.pkl')
        
        # Upload to model registry
        await self.upload_to_model_registry(
            'meetup_prediction',
            'v1',
            score
        )
    
    async def collect_training_data(self, start_date, end_date):
        """Collect labeled training data from database"""
        
        query = """
        SELECT 
            u.id,
            l.latitude,
            l.longitude,
            l.created_at,
            m.meeting_location_lat,
            m.meeting_location_lng
        FROM locations l
        JOIN users u ON l.user_id = u.id
        LEFT JOIN meetings m ON u.id = m.user_id
        WHERE l.created_at BETWEEN $1 AND $2
        AND m.meeting_location_lat IS NOT NULL
        """
        
        data = await self.db.fetch(query, start_date, end_date)
        return data
```

---

## 5. Real-Time Inference

### 5.1 Model Serving

```python
from fastapi import FastAPI
import joblib

app = FastAPI()

# Load model at startup
model = None

@app.on_event("startup")
async def load_model():
    global model
    model = joblib.load('models/meetup_prediction_v1.pkl')
    logger.info("Model loaded")

@app.post("/predict/meetup-location")
async def predict_meetup(
    user_ids: List[str],
    preferences: dict
):
    """Real-time meetup prediction endpoint"""
    
    service = MeetupPredictionService()
    prediction = await service.predict_meetup_location(
        user_ids,
        preferences
    )
    
    return prediction

@app.post("/detect/anomalies")
async def detect_anomalies(user_id: str):
    """Real-time anomaly detection"""
    
    service = AnomalyDetectionService()
    score = await service.calculate_abuse_score(user_id)
    
    if score['overall_score'] > 0.5:
        await alert_security_team(score)
    
    return score
```

---

## 6. Continuous Learning

### 6.1 Model Update Strategy

```python
class ContinuousLearningService:
    async def update_model_with_new_data(self):
        """Periodically update model with new data"""
        
        # Run daily
        schedule.every().day.at("02:00").do(
            self.retrain_models
        )
    
    async def retrain_models(self):
        """Retrain all models"""
        
        logger.info("Starting model retraining...")
        
        try:
            # Get new training data from last 7 days
            new_data = await self.collect_recent_data(days=7)
            
            # Train new model
            new_model = await self.train_meetup_model(new_data)
            
            # Evaluate against old model
            old_score = await self.evaluate_model('v1')
            new_score = await self.evaluate_model(new_model)
            
            # If better, deploy
            if new_score > old_score * 1.01:  # 1% improvement
                await self.deploy_model(new_model, 'v2')
                logger.info(f"New model deployed. Score: {new_score}")
            else:
                logger.info(f"New model not better. Old: {old_score}, New: {new_score}")
        
        except Exception as e:
            logger.error(f"Model retraining failed: {e}")
            await alert_ml_team()
```

---

**Complete AI/ML system with intelligent predictions, abuse detection, and continuous learning.**
