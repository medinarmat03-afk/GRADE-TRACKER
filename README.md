
SOURCECODE
import sqlite3

import os

import shutil

from datetime import datetime

from collections import deque


DATABASE_FILE = "grades.db"



def get_db_connection():

   # database connection

   db_path = os.path.join(os.path.dirname(__file__), DATABASE_FILE)

   conn = sqlite3.connect(db_path)

   conn.execute("PRAGMA foreign_keys = ON")

   return conn



def setup_database():

  # Initialize database

   conn = get_db_connection()

   cursor = conn.cursor()


   # Create students table

   cursor.execute("""CREATE TABLE IF NOT EXISTS students (

       id INTEGER PRIMARY KEY,

       student_id TEXT,

       course TEXT

   )""")


   # Create subjects table

   cursor.execute("""CREATE TABLE IF NOT EXISTS subjects (

       id INTEGER PRIMARY KEY AUTOINCREMENT,

       name TEXT UNIQUE,

       weights TEXT

   )""")


   # Create grades table

   cursor.execute("""CREATE TABLE IF NOT EXISTS grades (

       id INTEGER PRIMARY KEY AUTOINCREMENT,

       subject TEXT NOT NULL,

       activity_type TEXT NOT NULL,

       score REAL NOT NULL,

       total_score REAL NOT NULL,

       date_added TEXT NOT NULL

   )""")


   # Create attachments table

   cursor.execute("""CREATE TABLE IF NOT EXISTS attachments (

       id INTEGER PRIMARY KEY AUTOINCREMENT,

       subject TEXT NOT NULL,

       activity_type TEXT NOT NULL,

       file_name TEXT NOT NULL,

       file_path TEXT NOT NULL,

       uploaded_at TEXT NOT NULL

   )""")


   # Initialize default student record

   cursor.execute("INSERT OR IGNORE INTO students (id, student_id, course) VALUES (1, '---', '---')")


   conn.commit()

   conn.close()


# Undo and Redo Feature


# Undo/redo logic

class UndoRedoManager:

   # Manages undo/redo


   def __init__(self, max_history=50):

       self.undo_stack = deque(maxlen=max_history)

       self.redo_stack = deque(maxlen=max_history)


   def add_action(self, action):

       # Action for undo and redo

       self.undo_stack.append(action)

       self.redo_stack.clear()

       print(f"[UNDO] Action added: {action.__class__.__name__} (Stack size: {len(self.undo_stack)})")


   def undo(self):

       # Undo the last action

       if not self.undo_stack:

           print("[UNDO] Nothing to undo")

           return False


       action = self.undo_stack.pop()

       print(f"[UNDO] Undoing: {action.__class__.__name__}")

       try:

           action.undo()

           self.redo_stack.append(action)

           print(f"[UNDO] Success! Redo stack size: {len(self.redo_stack)}")

           return True

       except Exception as e:

           print(f"[UNDO] Error: {e}")

           self.undo_stack.append(action)

           return False


   def redo(self):

       # Redo the last undone action

       if not self.redo_stack:

           print("[REDO] Nothing to redo")

           return False


       action = self.redo_stack.pop()

       print(f"[REDO] Redoing: {action.__class__.__name__}")

       try:

           action.redo()

           self.undo_stack.append(action)

           print(f"[REDO] Success! Undo stack size: {len(self.undo_stack)}")

           return True

       except Exception as e:

           print(f"[REDO] Error: {e}")

           self.redo_stack.append(action)

           return False


   def can_undo(self):

       # Check if undo is available

       return len(self.undo_stack) > 0


   def can_redo(self):

       # Check if redo is available

       return len(self.redo_stack) > 0


   def clear(self):

       # Clear all history

       self.undo_stack.clear()

       self.redo_stack.clear()

       print("[UNDO/REDO] History cleared")


   def get_undo_count(self):

       # Get number of undo action

       return len(self.undo_stack)


   def get_redo_count(self):

       # Get number of redo actions available

       return len(self.redo_stack)


   def get_undo_message(self):

       # Get description of last undo action

       if self.undo_stack:

           return self.undo_stack[-1].get_description()

       return "Nothing to undo"


   def get_redo_message(self):

       # Get description of last redo action

       if self.redo_stack:

           return self.redo_stack[-1].get_description()

       return "Nothing to redo"


   def get_undo_history(self):

       # Get list of all undo actions

       return [action.get_description() for action in reversed(self.undo_stack)]


   def get_redo_history(self):

       # Get list of all redo actions

       return [action.get_description() for action in reversed(self.redo_stack)]



# Global undo/redo manager instance

undo_manager = UndoRedoManager()



# UndoRedo action

class Action:

   # Base class for actions like undo and redo


   def undo(self):

       raise NotImplementedError


   def redo(self):

       raise NotImplementedError


   def get_description(self):

       return self.__class__.__name__



class AddGradeAction(Action):

   # action for adding a grade"""


   def __init__(self, grade_id, subject, activity_type, score, total_score, date_added):

       self.grade_id = grade_id

       self.subject = subject

       self.activity_type = activity_type

       self.score = score

       self.total_score = total_score

       self.date_added = date_added


   def undo(self):

       # Remove the grade

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("DELETE FROM grades WHERE id=?", (self.grade_id,))

       conn.commit()

       conn.close()

       print(f"[UNDO] Deleted grade ID {self.grade_id}")


   def redo(self):

       # Readded the grade

       conn = get_db_connection()

       cursor = conn.cursor()

       try:

           

           cursor.execute(

               "INSERT INTO grades (id, subject, activity_type, score, total_score, date_added) VALUES (?, ?, ?, ?, ?, ?)",

               (self.grade_id, self.subject, self.activity_type, float(self.score), float(self.total_score),

                self.date_added)

           )

       except Exception as e:

           

           print(f"[REDO] Inserting with auto ID: {e}")

           cursor.execute(

               "INSERT INTO grades (subject, activity_type, score, total_score, date_added) VALUES (?, ?, ?, ?, ?)",

               (self.subject, self.activity_type, float(self.score), float(self.total_score), self.date_added)

           )

           self.grade_id = cursor.lastrowid

       conn.commit()

       conn.close()

       print(f"[REDO] Re-added grade ID {self.grade_id}")


   def get_description(self):

       return f"Add grade: {self.subject} - {self.activity_type} ({self.score}/{self.total_score})"



class DeleteGradeAction(Action):

   #Delete Grade


   def __init__(self, grade_id, subject, activity_type, score, total_score, date_added):

       self.grade_id = grade_id

       self.subject = subject

       self.activity_type = activity_type

       self.score = score

       self.total_score = total_score

       self.date_added = date_added


   def undo(self):

       # Undelete the grade

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute(

           "INSERT INTO grades (id, subject, activity_type, score, total_score, date_added) VALUES (?, ?, ?, ?, ?, ?)",

           (self.grade_id, self.subject, self.activity_type, float(self.score), float(self.total_score),

            self.date_added)

       )

       conn.commit()

       conn.close()

       print(f"[UNDO] Restored grade ID {self.grade_id}")


   def redo(self):

       # Redelete the grade

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("DELETE FROM grades WHERE id=?", (self.grade_id,))

       conn.commit()

       conn.close()

       print(f"[REDO] Re-deleted grade ID {self.grade_id}")


   def get_description(self):

       return f"Delete grade: {self.subject} - {self.activity_type}"



class AddSubjectAction(Action):

   # Action for adding a subject


   def __init__(self, subject_id, name, weights):

       self.subject_id = subject_id

       self.name = name

       self.weights = weights


   def undo(self):

       # Remove the subject

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("DELETE FROM subjects WHERE id=?", (self.subject_id,))

       conn.commit()

       conn.close()

       print(f"[UNDO] Deleted subject '{self.name}'")


   def redo(self):

       """Re-add the subject"""

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute(

           "INSERT INTO subjects (id, name, weights) VALUES (?, ?, ?)",

           (self.subject_id, self.name, self.weights)

       )

       conn.commit()

       conn.close()

       print(f"[REDO] Re-added subject '{self.name}'")


   def get_description(self):

       return f"Add subject: {self.name}"



class DeleteSubjectAction(Action):

   # Action for deleting a subject


   def __init__(self, subject_id, name, weights, related_grades, related_attachments):

       self.subject_id = subject_id

       self.name = name

       self.weights = weights

       self.related_grades = related_grades

       self.related_attachments = related_attachments


   def undo(self):

       # Restore the subject and all related data

       conn = get_db_connection()

       cursor = conn.cursor()


       # Restore subject

       cursor.execute(

           "INSERT INTO subjects (id, name, weights) VALUES (?, ?, ?)",

           (self.subject_id, self.name, self.weights)

       )


       # Restore grades

       for grade in self.related_grades:

           # grade is a tuple: (id, subject, activity_type, score, total_score, date_added)

           cursor.execute(

               "INSERT INTO grades (id, subject, activity_type, score, total_score, date_added) VALUES (?, ?, ?, ?, ?, ?)",

               (grade[0], grade[1], grade[2], float(grade[3]), float(grade[4]), grade[5])

           )


       # Restore attachments

       for attachment in self.related_attachments:

           cursor.execute(

               "INSERT INTO attachments (id, subject, activity_type, file_name, file_path, uploaded_at) VALUES (?, ?, ?, ?, ?, ?)",

               attachment

           )


       conn.commit()

       conn.close()

       print(f"[UNDO] Restored subject '{self.name}'")


   def redo(self):

       # Redelete the subject and related data

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("DELETE FROM subjects WHERE id=?", (self.subject_id,))

       cursor.execute("DELETE FROM grades WHERE subject=?", (self.name,))

       cursor.execute("DELETE FROM attachments WHERE subject=?", (self.name,))

       conn.commit()

       conn.close()

       print(f"[REDO] Re-deleted subject '{self.name}'")


   def get_description(self):

       return f"Delete subject: {self.name}"



class EditSubjectAction(Action):

   # Action for editing a subject


   def __init__(self, subject_id, old_name, new_name, old_weights, new_weights):

       self.subject_id = subject_id

       self.old_name = old_name

       self.new_name = new_name

       self.old_weights = old_weights

       self.new_weights = new_weights


   def undo(self):

       # Restore subject

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("UPDATE subjects SET name=?, weights=? WHERE id=?",

                      (self.old_name, self.old_weights, self.subject_id))

       cursor.execute("UPDATE grades SET subject=? WHERE subject=?",

                      (self.old_name, self.new_name))

       cursor.execute("UPDATE attachments SET subject=? WHERE subject=?",

                      (self.old_name, self.new_name))

       conn.commit()

       conn.close()

       print(f"[UNDO] Restored subject to '{self.old_name}'")


   def redo(self):

       #Reedit

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("UPDATE subjects SET name=?, weights=? WHERE id=?",

                      (self.new_name, self.new_weights, self.subject_id))

       cursor.execute("UPDATE grades SET subject=? WHERE subject=?",

                      (self.new_name, self.old_name))

       cursor.execute("UPDATE attachments SET subject=? WHERE subject=?",

                      (self.new_name, self.old_name))

       conn.commit()

       conn.close()

       print(f"[REDO] Edited subject to '{self.new_name}'")


   def get_description(self):

       return f"Edit subject: {self.old_name} → {self.new_name}"



#Student Service

class StudentService:

   # Student profile management


   @staticmethod

   def get_student_profile():

      # Get student profile

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("SELECT student_id, course FROM students WHERE id=1")

       row = cursor.fetchone()

       conn.close()

       return row


   @staticmethod

   def save_student_profile(sid, course):

       # Save student profile

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("UPDATE students SET student_id=?, course=? WHERE id=1", (sid, course))

       conn.commit()

       conn.close()



# SubjectService

class SubjectService:

   # Subject management


   @staticmethod

   def add_subject(name, weights=""):

       # Add Subject

       try:

           conn = get_db_connection()

           cursor = conn.cursor()

           cursor.execute("INSERT INTO subjects (name, weights) VALUES (?, ?)", (name, weights))

           conn.commit()


           # Get the ID of the inserted subject

           cursor.execute("SELECT id FROM subjects WHERE name=?", (name,))

           subject_id = cursor.fetchone()[0]

           conn.close()


           # Add to undo

           action = AddSubjectAction(subject_id, name, weights)

           undo_manager.add_action(action)

           return True

       except Exception as e:

           print(f"Error adding subject: {e}")

           return False


   @staticmethod

   def get_all_subjects():

       # Get all subjects

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("SELECT id, name, weights FROM subjects ORDER BY name")

       rows = cursor.fetchall()

       conn.close()

       return rows


   @staticmethod

   def get_subject_names():

       # Get list of subject names

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("SELECT name FROM subjects ORDER BY name")

       rows = cursor.fetchall()

       conn.close()

       return [row[0] for row in rows]


   @staticmethod

   def update_subject(subject_id, new_name, new_weights, old_name):

       # Update a subject

       try:

           conn = get_db_connection()

           cursor = conn.cursor()


           # Get old data for undo

           cursor.execute("SELECT weights FROM subjects WHERE id=?", (subject_id,))

           old_weights = cursor.fetchone()[0]


           cursor.execute("UPDATE subjects SET name=?, weights=? WHERE id=?",

                          (new_name, new_weights, subject_id))

           cursor.execute("UPDATE grades SET subject=? WHERE subject=?",

                          (new_name, old_name))

           cursor.execute("UPDATE attachments SET subject=? WHERE subject=?",

                          (new_name, old_name))

           conn.commit()

           conn.close()


           # Add to undo stack

           action = EditSubjectAction(subject_id, old_name, new_name, old_weights, new_weights)

           undo_manager.add_action(action)

       except Exception as e:

           print(f"Error updating subject: {e}")


   @staticmethod

   def delete_subject(subject_id, name):

       # Delete subject

       try:

           conn = get_db_connection()

           cursor = conn.cursor()


           # Get data for undo

           cursor.execute("SELECT weights FROM subjects WHERE id=?", (subject_id,))

           weights = cursor.fetchone()[0]


           # Get related grades

           cursor.execute(

               "SELECT id, subject, activity_type, score, total_score, date_added FROM grades WHERE subject=?",

               (name,))

           related_grades = cursor.fetchall()


           # Get related attachments

           cursor.execute(

               "SELECT id, subject, activity_type, file_name, file_path, uploaded_at FROM attachments WHERE subject=?",

               (name,))

           related_attachments = cursor.fetchall()


           cursor.execute("DELETE FROM subjects WHERE id=?", (subject_id,))

           cursor.execute("DELETE FROM grades WHERE subject=?", (name,))

           cursor.execute("DELETE FROM attachments WHERE subject=?", (name,))

           conn.commit()

           conn.close()


           # Add to undo stack

           action = DeleteSubjectAction(subject_id, name, weights, related_grades, related_attachments)

           undo_manager.add_action(action)

       except Exception as e:

           print(f"Error deleting subject: {e}")



# GradeService

class GradeService:

   # Grade management


   @staticmethod

   def add_grade(subject, activity_type, score, total_score):

       # Add a new grade

       try:

           conn = get_db_connection()

           cursor = conn.cursor()


           # Ensure tables exist

           cursor.execute("""CREATE TABLE IF NOT EXISTS grades (

               id INTEGER PRIMARY KEY AUTOINCREMENT,

               subject TEXT NOT NULL,

               activity_type TEXT NOT NULL,

               score REAL NOT NULL,

               total_score REAL NOT NULL,

               date_added TEXT NOT NULL

           )""")


           # Simple timestamp

           date_added = datetime.now().strftime("%Y-%m-%d %H:%M:%S")


           # Insert grade

           cursor.execute(

               "INSERT INTO grades (subject, activity_type, score, total_score, date_added) VALUES (?, ?, ?, ?, ?)",

               (subject, activity_type, float(score), float(total_score), date_added)

           )


           # Get the ID of the inserted grade

           grade_id = cursor.lastrowid


           conn.commit()

           conn.close()


           # Add to undo stack

           action = AddGradeAction(grade_id, subject, activity_type, score, total_score, date_added)

           undo_manager.add_action(action)

           return True


       except Exception as e:

           print(f"Database Error in add_grade: {e}")

           raise


   @staticmethod

   def get_all_grades(filter_value=None):

       # Get all grades

       conn = get_db_connection()

       cursor = conn.cursor()


       if filter_value:

           cursor.execute(

               "SELECT id, subject, activity_type, score, total_score FROM grades WHERE subject=? ORDER BY date_added DESC",

               (filter_value,)

           )

       else:

           cursor.execute("SELECT id, subject, activity_type, score, total_score FROM grades ORDER BY date_added DESC")


       rows = cursor.fetchall()

       conn.close()

       return rows


   @staticmethod

   def delete_grade(grade_id):

       # Delete grade

       try:

           conn = get_db_connection()

           cursor = conn.cursor()


           # Get grade data for undo

           cursor.execute("SELECT subject, activity_type, score, total_score, date_added FROM grades WHERE id=?",

                          (grade_id,))

           grade_data = cursor.fetchone()


           if grade_data:

               subject, activity_type, score, total_score, date_added = grade_data

               cursor.execute("DELETE FROM grades WHERE id=?", (grade_id,))

               conn.commit()

               conn.close()


               # Add to undo stack

               action = DeleteGradeAction(grade_id, subject, activity_type, score, total_score, date_added)

               undo_manager.add_action(action)

           else:

               conn.close()

       except Exception as e:

           print(f"Error deleting grade: {e}")


   @staticmethod

   def get_recent_grades(limit=50):

       # Get recent grades

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("SELECT subject, score, total_score FROM grades ORDER BY date_added DESC LIMIT ?", (limit,))

       rows = cursor.fetchall()

       conn.close()

       return rows



#STATISTICS SERVICE

class StatisticsService:

   # Statistics and analytics


   @staticmethod

   def compute_weighted_grade(subject_name, weights_string=""):

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("SELECT activity_type, score, total_score FROM grades WHERE subject=?", (subject_name,))

       records = cursor.fetchall()

       conn.close()


       if not records:

           return 0.0


       # 1. Parse weights into {'quiz': 30.0, 'exam': 50.0}

       weight_map = {}

       if weights_string:

           try:

               pairs = weights_string.split(',')

               for pair in pairs:

                   if ':' in pair:

                       key, val = pair.split(':')

                       weight_map[key.strip().lower()] = float(val.strip())

           except (ValueError, AttributeError):

               weight_map = {}


       # 2. Group averages by category

       groups = {}

       for atype, score, total in records:

           ntype = atype.strip().lower()  # Normalize name

           if ntype not in groups:

               groups[ntype] = []

           if total > 0:

               groups[ntype].append(float(score) / float(total))


       # 3. Calculate Weighted Average

       if weight_map:

           final_grade = 0.0

           for atype, weight in weight_map.items():

               if atype in groups:

                   # Average for this category (e.g., 0.85)

                   category_avg = sum(groups[atype]) / len(groups[atype])


                   # Formula: (Average * 100) * (Weight / 100)

                   # Example: (0.85 * 100) * (30 / 100) = 85 * 0.30 = 25.5

                   contribution = (category_avg * 100.0) * (weight / 100.0)

                   final_grade += contribution


           return round(final_grade, 2)


       else:

           # Simple average if no weights provided

           sum_score = sum(float(r[1]) for r in records)

           sum_total = sum(float(r[2]) for r in records)

           return round((sum_score / sum_total * 100.0), 2) if sum_total > 0 else 0.0




   @staticmethod

   @staticmethod

   def get_dashboard_stats():

       # Get statistics for the dashboard

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("SELECT COUNT(*) FROM subjects")

       subj_count = cursor.fetchone()[0]

       cursor.execute("SELECT name, weights FROM subjects")

       subjects = cursor.fetchall()

       conn.close()


       total_avg, total_gpa, count = 0, 0, 0

       for name, weights in subjects:

           avg = StatisticsService.compute_weighted_grade(name, weights)

           if avg > 0:

               total_avg += avg


               # Convert to PUP grading scale

               if avg >= 97:

                   gpa = 1.0

               elif avg >= 94:

                   gpa = 1.25

               elif avg >= 91:

                   gpa = 1.50

               elif avg >= 88:

                   gpa = 1.75

               elif avg >= 85:

                   gpa = 2.0

               elif avg >= 82:

                   gpa = 2.25

               elif avg >= 79:

                   gpa = 2.50

               elif avg >= 76:

                   gpa = 2.75

               elif avg >= 75:

                   gpa = 3.0

               else:

                   gpa = 5.0


               total_gpa += gpa

               count += 1


       return {

           "subject_count": subj_count,

           "overall_average": round(total_avg / count, 2) if count > 0 else 0.0,

           "gpa": round(total_gpa / count, 2) if count > 0 else 0.0

       }


   @staticmethod

   def get_formula_breakdown(subject_name, weights_str):

   

       weight_map = {}

       if weights_str:

           try:

               # Ex: Quiz:20, Exam:80" into ['Quiz:20', ' Exam:80']

               pairs = weights_str.split(',')

               for pair in pairs:

                   if ':' in pair:

                       name, val = pair.split(':')

                      # the weight has to match in order for its % to be recognized

                       weight_map[name.strip().lower()] = float(val.strip())

           except ValueError:

               pass  # Or handle error for badly formatted strings


       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute("SELECT activity_type, score, total_score FROM grades WHERE subject=?", (subject_name,))

       grades = cursor.fetchall()

       conn.close()


       if not grades:

           return None


       # Group by activity type

       activities = {}

       for activity_type, score, total_score in grades:

           activities.setdefault(activity_type, []).append((score, total_score))


       breakdown = []

       for activity, scores in activities.items():

           total_score = sum(s for s, _ in scores)

           total_total = sum(t for _, t in scores)


           weight = weight_map.get(activity.strip().lower(), 0)


           if total_total > 0:

               average = (total_score / total_total) * 100

               contribution = average * (weight / 100)

           else:

               average = 0

               contribution = 0


           breakdown.append({

               'category': activity,

               'average': f"{average:.2f}",

               'weight_percentage': f"{weight}%",

               'contribution': f"{contribution:.2f}"

           })


       return breakdown


   @staticmethod

   def get_progress_trend(subject_name=None):

       # Returns the average grade trend over time

       conn = get_db_connection()

       cursor = conn.cursor()


       query = "SELECT date_added, score, total_score FROM grades"

       params = ()

       if subject_name:

           query += " WHERE subject=? "

           params = (subject_name,)

       query += " ORDER BY date_added ASC"


       cursor.execute(query, params)

       records = cursor.fetchall()

       conn.close()


       trend = []

       running_sum_score = 0

       running_sum_total = 0


       for date, score, total in records:

           running_sum_score += score

           running_sum_total += total

           current_avg = (running_sum_score / running_sum_total) * 100 if running_sum_total > 0 else 0

           trend.append({

               "date": date.split(' ')[0] if date else "N/A",

               "average": round(current_avg, 2)

           })

       return trend



# Attachment

class AttachmentService:

   @staticmethod

   def save_attachment(subject, activity_type, file_path):

       # Save file attachment

       try:

           if not os.path.exists(file_path):

               return False


           # Create uploads directory

           uploads_dir = os.path.join(os.path.dirname(__file__), "uploads")

           if not os.path.exists(uploads_dir):

               os.makedirs(uploads_dir)


           # Copy file

           file_name = os.path.basename(file_path)

           dest_path = os.path.join(uploads_dir, file_name)

           shutil.copy(file_path, dest_path)


           # Save to database

           conn = get_db_connection()

           cursor = conn.cursor()

           uploaded_at = datetime.now().strftime("%Y-%m-%d %H:%M:%S")


           cursor.execute(

               "INSERT INTO attachments (subject, activity_type, file_name, file_path, uploaded_at) VALUES (?, ?, ?, ?, ?)",

               (subject, activity_type, file_name, dest_path, uploaded_at)

           )


           conn.commit()

           conn.close()

           return True


       except Exception as e:

           print(f"Error saving attachment: {e}")

           return False


   @staticmethod

   def get_all_attachments():

       #  Get all attachments

       conn = get_db_connection()

       cursor = conn.cursor()

       cursor.execute(

           "SELECT id, subject, activity_type, file_name, file_path, uploaded_at FROM attachments ORDER BY uploaded_at DESC")

       rows = cursor.fetchall()

       conn.close()


       return [

           {

               'id': row[0],

               'subject': row[1],

               'activity_type': row[2],

               'file_name': row[3],

               'file_path': row[4],

               'uploaded_at': row[5]

           }

           for row in rows

       ]


   @staticmethod

   def delete_attachment(att_id, file_path):

       # Delete attachment

       try:

           # Delete file

           if os.path.exists(file_path):

               os.remove(file_path)


           # Delete from database

           conn = get_db_connection()

           cursor = conn.cursor()

           cursor.execute("DELETE FROM attachments WHERE id=?", (att_id,))

           conn.commit()

           conn.close()

           return True


       except Exception as e:

           print(f"Error deleting attachment: {e}")

           return False





FRONT END




import tkinter as tk

from tkinter import messagebox, ttk, filedialog

from engine2 import (StudentService, SubjectService, GradeService, StatisticsService, AttachmentService, undo_manager)

from login_system import show_login, AuthService

from datetime import datetime

import os

import subprocess

import platform

import sys



# Color Pallete

COLORS = {

   'primary': '#DC143C',

   'primary_dark': '#B91C1C',

   'accent': '#FEE2E2',

   'white': '#FFFFFF',

   'bg': '#F9FAFB',

   'text': '#1F2937',

   'text_light': '#6B7280',

   'success': '#10B981',

   'danger': '#EF4444'

}



class ModernButton(tk.Button):


   def __init__(self, parent, text, command=None, style='primary', **kwargs):

       colors = {

           'primary': (COLORS['primary'], COLORS['white']),

           'secondary': (COLORS['white'], COLORS['primary']),

           'success': (COLORS['success'], COLORS['white']),

           'danger': (COLORS['danger'], COLORS['white'])

       }

       bg, fg = colors.get(style, colors['primary'])


       super().__init__(

           parent, text=text, command=command,

           bg=bg, fg=fg, font=('Segoe UI', 10, 'bold'),

           relief='flat', cursor='hand2',

           padx=20, pady=10, **kwargs

       )

       self.bind('<Enter>', lambda e: self.config(bg=self.lighten_color(bg)))

       self.bind('<Leave>', lambda e: self.config(bg=bg))


   def lighten_color(self, color):

       if color == COLORS['primary']:

           return COLORS['primary_dark']

       return color



class ModernEntry(tk.Frame):

   


   def __init__(self, parent, label, **kwargs):

       super().__init__(parent, bg=COLORS['white'])


       tk.Label(

           self, text=label, bg=COLORS['white'],

           fg=COLORS['text'], font=('Segoe UI', 9, 'bold')

       ).pack(anchor='w', pady=(0, 5))


       self.entry = tk.Entry(

           self, font=('Segoe UI', 10), relief='flat',

           borderwidth=0, **kwargs

       )

       self.entry.pack(fill='x', ipady=8)


   def get(self):

       return self.entry.get()


   def insert(self, index, string):

       self.entry.insert(index, string)



class Card(tk.Frame):

   

   def __init__(self, parent, **kwargs):

       super().__init__(

           parent, bg=COLORS['white'],

           relief='flat', borderwidth=0,

           **kwargs

       )



class GradeTrackerApp:

   def __init__(self, root, user_data):

       # Initialize data from the login

       self.root = root

       self.user_data = user_data


       self.root.title("PUP Grade Tracker")

       self.root.geometry("1200x700")

       self.root.configure(bg=COLORS['bg'])


       self.pages = {

           'dashboard': self.create_dashboard,

           'subjects': self.create_subjects,

           'grades': self.create_grades,

           'analytics': self.create_analytics,

           'grade_heat_map': self.create_grade_heat_map,

           'transparency': self.create_transparency,

           'attachments': self.create_attachments,

           'progress': self.create_progress_tracking,

           'history': self.create_history_page  # NEW: Undo/Redo page

       }


       self.create_header()

       self.create_sidebar()

       self.create_main_content()

       self.show_page('dashboard')


   def create_header(self):

       #Header bar with user info and logout

       header = tk.Frame(self.root, bg=COLORS['primary'], height=80)

       header.pack(fill='x')

       header.pack_propagate(False)


       try:

           from PIL import Image, ImageTk

           logo_path = os.path.join(os.path.dirname(__file__), "LOGO.png")

           img = Image.open(logo_path)

           img = img.resize((50, 50), Image.Resampling.LANCZOS)

           self.logo_img = ImageTk.PhotoImage(img)

           logo_label = tk.Label(header, image=self.logo_img, bg=COLORS['primary'])

           logo_label.pack(side='left', padx=(30, 10), pady=15)

       except:

           tk.Label(

               header, text="🎓", font=('Segoe UI', 24),

               bg=COLORS['primary']

           ).pack(side='left', padx=(30, 10))


       tk.Label(

           header, text="Polytechnic University of The Philippines",

           font=('Segoe UI', 16, 'bold'),

           fg=COLORS['white'], bg=COLORS['primary']

       ).pack(side='left', pady=20)


       # User info and logout

       user_info_frame = tk.Frame(header, bg=COLORS['primary'])

       user_info_frame.pack(side='right', padx=30, pady=20)


       user_display_text = f"👤 {self.user_data['full_name']}"

       tk.Label(

           user_info_frame, text=user_display_text,

           font=('Segoe UI', 10),

           fg=COLORS['white'], bg=COLORS['primary']

       ).pack(side='left', padx=(0, 20))


       ModernButton(

           user_info_frame, "🚪 Logout",

           command=self.logout_user, style='secondary'

       ).pack(side='left')


   def logout_user(self):

       # Handle user logout

       if messagebox.askyesno("Logout", "Are you sure you want to logout?"):

           for widget in self.root.winfo_children():

               widget.destroy()

           show_login(self.root, self.on_login_success)


   def on_login_success(self, user_data):

       # Callback for re-login

       self.user_data = user_data

       GradeTrackerApp(self.root, user_data)


   def create_sidebar(self):

       # Left navigation sidebar

       sidebar = tk.Frame(self.root, bg=COLORS['white'], width=200)

       sidebar.pack(side='left', fill='y')

       sidebar.pack_propagate(False)


       nav_items = [

           ('🏠 Dashboard', 'dashboard'),

           ('📚 Subjects', 'subjects'),

           ('📝 Grades', 'grades'),

           ('📊 Analytics', 'analytics'),

           ('📎 Attachments', 'attachments'),

           ('🗺️ Heat Map', 'grade_heat_map'),

           ('👁 Transparency', 'transparency'),

           ('📈 Progress', 'progress'),

           ('⏮️ History', 'history')  # NEW: Undo/Redo menu item

       ]


       for text, page in nav_items:

           btn = tk.Button(

               sidebar, text=text, font=('Segoe UI', 11),

               bg=COLORS['white'], fg=COLORS['text'],

               relief='flat', anchor='w', padx=25, pady=15,

               cursor='hand2', command=lambda p=page: self.show_page(p)

           )

           btn.pack(fill='x')

           btn.bind('<Enter>', lambda e, b=btn: b.config(bg=COLORS['accent']))

           btn.bind('<Leave>', lambda e, b=btn: b.config(bg=COLORS['white']))


   def create_main_content(self):

       self.content = tk.Frame(self.root, bg=COLORS['bg'])

       self.content.pack(side='right', fill='both', expand=True, padx=20, pady=20)


   def show_page(self, page_name):

       # Clear and show selected page

       for widget in self.content.winfo_children():

           widget.destroy()

       self.pages[page_name]()


   def create_history_page(self):

       # Undo/Redo History page

       tk.Label(

           self.content, text="Action History (Undo/Redo)",

           font=('Segoe UI', 18, 'bold'),

           bg=COLORS['bg'], fg=COLORS['text']

       ).pack(anchor='w', pady=(0, 20))


       # Stats Card

       stats_card = Card(self.content)

       stats_card.pack(fill='x', pady=(0, 20))


       stats_frame = tk.Frame(stats_card, bg=COLORS['white'])

       stats_frame.pack(fill='x', padx=20, pady=15)


       undo_count = undo_manager.get_undo_count()

       redo_count = undo_manager.get_redo_count()


       tk.Label(

           stats_frame, text=f"📋 Undo Available: {undo_count}",

           font=('Segoe UI', 11, 'bold'),

           bg=COLORS['white'], fg=COLORS['primary']

       ).pack(side='left', padx=20)


       tk.Label(

           stats_frame, text=f"🔄 Redo Available: {redo_count}",

           font=('Segoe UI', 11, 'bold'),

           bg=COLORS['white'], fg=COLORS['success']

       ).pack(side='left', padx=20)


       # Action Buttons

       btn_frame = tk.Frame(self.content, bg=COLORS['bg'])

       btn_frame.pack(fill='x', pady=(0, 20))


       def perform_undo():

           if undo_manager.undo():

               messagebox.showinfo("Success", "✅ Action undone successfully!")

               self.show_page('history')

           else:

               messagebox.showwarning("No Undo", "Nothing to undo!")


       def perform_redo():

           if undo_manager.redo():

               messagebox.showinfo("Success", "✅ Action redone successfully!")

               self.show_page('history')

           else:

               messagebox.showwarning("No Redo", "Nothing to redo!")


       def clear_history():

           if messagebox.askyesno("Clear History", "Clear all undo/redo history? This cannot be undone."):

               undo_manager.clear()

               messagebox.showinfo("Cleared", "History cleared successfully!")

               self.show_page('history')


       undo_btn = ModernButton(btn_frame, "⏮️ Undo Last Action", command=perform_undo, style='primary')

       undo_btn.pack(side='left', padx=5)

       if not undo_manager.can_undo():

           undo_btn.config(state='disabled')


       redo_btn = ModernButton(btn_frame, "⏭️ Redo Last Action", command=perform_redo, style='success')

       redo_btn.pack(side='left', padx=5)

       if not undo_manager.can_redo():

           redo_btn.config(state='disabled')


       ModernButton(btn_frame, "🗑️ Clear History", command=clear_history, style='danger').pack(side='left', padx=5)


       # History Lists

       history_container = tk.Frame(self.content, bg=COLORS['bg'])

       history_container.pack(fill='both', expand=True)


       # Undo History

       undo_card = Card(history_container)

       undo_card.pack(side='left', fill='both', expand=True, padx=(0, 10))


       tk.Label(

           undo_card, text="📋 Undo Stack (Most Recent First)",

           font=('Segoe UI', 12, 'bold'),

           bg=COLORS['white'], fg=COLORS['text']

       ).pack(anchor='w', padx=20, pady=(15, 10))


       undo_list_container = tk.Frame(undo_card, bg=COLORS['white'])

       undo_list_container.pack(fill='both', expand=True, padx=20, pady=(0, 20))


       undo_canvas = tk.Canvas(undo_list_container, bg=COLORS['white'], highlightthickness=0)

       undo_scrollbar = tk.Scrollbar(undo_list_container, orient="vertical", command=undo_canvas.yview)

       undo_scrollable = tk.Frame(undo_canvas, bg=COLORS['white'])


       undo_scrollable.bind(

           "<Configure>",

           lambda e: undo_canvas.configure(scrollregion=undo_canvas.bbox("all"))

       )


       undo_canvas.create_window((0, 0), window=undo_scrollable, anchor="nw")

       undo_canvas.configure(yscrollcommand=undo_scrollbar.set)


       undo_canvas.pack(side="left", fill="both", expand=True)

       undo_scrollbar.pack(side="right", fill="y")


       undo_history = undo_manager.get_undo_history()

       if not undo_history:

           tk.Label(

               undo_scrollable, text="No undo actions available",

               bg=COLORS['white'], fg=COLORS['text_light'],

               font=('Segoe UI', 10)

           ).pack(pady=20)

       else:

           for i, action_str in enumerate(undo_history):

               item_frame = tk.Frame(undo_scrollable, bg=COLORS['accent'])

               item_frame.pack(fill='x', pady=2, padx=5)


               tk.Label(

                   item_frame, text=f"{i+1}. {action_str}",

                   bg=COLORS['accent'], fg=COLORS['text'],

                   font=('Segoe UI', 9), anchor='w'

               ).pack(fill='x', padx=10, pady=8)


       # Redo History

       redo_card = Card(history_container)

       redo_card.pack(side='right', fill='both', expand=True, padx=(10, 0))


       tk.Label(

           redo_card, text="🔄 Redo Stack (Most Recent First)",

           font=('Segoe UI', 12, 'bold'),

           bg=COLORS['white'], fg=COLORS['text']

       ).pack(anchor='w', padx=20, pady=(15, 10))


       redo_list_container = tk.Frame(redo_card, bg=COLORS['white'])

       redo_list_container.pack(fill='both', expand=True, padx=20, pady=(0, 20))


       redo_canvas = tk.Canvas(redo_list_container, bg=COLORS['white'], highlightthickness=0)

       redo_scrollbar = tk.Scrollbar(redo_list_container, orient="vertical", command=redo_canvas.yview)

       redo_scrollable = tk.Frame(redo_canvas, bg=COLORS['white'])


       redo_scrollable.bind(

           "<Configure>",

           lambda e: redo_canvas.configure(scrollregion=redo_canvas.bbox("all"))

       )


       redo_canvas.create_window((0, 0), window=redo_scrollable, anchor="nw")

       redo_canvas.configure(yscrollcommand=redo_scrollbar.set)


       redo_canvas.pack(side="left", fill="both", expand=True)

       redo_scrollbar.pack(side="right", fill="y")


       redo_history = undo_manager.get_redo_history()

       if not redo_history:

           tk.Label(

               redo_scrollable, text="No redo actions available",

               bg=COLORS['white'], fg=COLORS['text_light'],

               font=('Segoe UI', 10)

           ).pack(pady=20)

       else:

           for i, action_str in enumerate(redo_history):

               item_frame = tk.Frame(redo_scrollable, bg='#E0F2FE')

               item_frame.pack(fill='x', pady=2, padx=5)


               tk.Label(

                   item_frame, text=f"{i+1}. {action_str}",

                   bg='#E0F2FE', fg=COLORS['text'],

                   font=('Segoe UI', 9), anchor='w'

               ).pack(fill='x', padx=10, pady=8)


   def create_dashboard(self):

       # Dashboard page

       stats_frame = tk.Frame(self.content, bg=COLORS['bg'])

       stats_frame.pack(fill='x', pady=(0, 20))


       stats = StatisticsService.get_dashboard_stats()

       student = StudentService.get_student_profile()


       sid = student[0] if student and student[0] != '---' else '---'

       course = student[1] if student and student[1] != '---' else '---'


       if sid == '---' and self.user_data['student_id']:

           sid = self.user_data['student_id']

       if course == '---' and self.user_data['course']:

           course = self.user_data['course']


       cards_data = [

           ('Student ID', sid, '👤'),

           ('Course', course, '🎓'),

           ('Subjects', stats['subject_count'], '📚'),

           ('Average', f"{stats['overall_average']}%", '⭐'),

           ('GPA (5.0)', stats['gpa'], '📈')

       ]


       for i, (label, value, icon) in enumerate(cards_data):

           card = Card(stats_frame)

           card.grid(row=0, column=i, padx=5, sticky='ew')

           stats_frame.columnconfigure(i, weight=1)


           tk.Label(

               card, text=icon, font=('Segoe UI', 24),

               bg=COLORS['white']

           ).pack(pady=(15, 5))


           tk.Label(

               card, text=str(value), font=('Segoe UI', 18, 'bold'),

               fg=COLORS['primary'], bg=COLORS['white']

           ).pack()


           tk.Label(

               card, text=label, font=('Segoe UI', 9),

               fg=COLORS['text_light'], bg=COLORS['white']

           ).pack(pady=(0, 15))


       ModernButton(

           self.content, "Edit Profile",

           command=self.show_profile_dialog

       ).pack(pady=10)


       activity_card = Card(self.content)

       activity_card.pack(fill='both', expand=True, pady=10)


       tk.Label(

           activity_card, text="Recent Activity",

           font=('Segoe UI', 14, 'bold'),

           bg=COLORS['white'], fg=COLORS['text']

       ).pack(anchor='w', padx=20, pady=(20, 10))


       container = tk.Frame(activity_card, bg=COLORS['white'])

       container.pack(fill='both', expand=True, padx=20, pady=(0, 20))


       canvas = tk.Canvas(container, bg=COLORS['white'], highlightthickness=0)

       scrollbar = tk.Scrollbar(container, orient="vertical", command=canvas.yview)

       scrollable_frame = tk.Frame(canvas, bg=COLORS['white'])


       scrollable_frame.bind(

           "<Configure>",

           lambda e: canvas.configure(scrollregion=canvas.bbox("all"))

       )


       canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")

       canvas.configure(yscrollcommand=scrollbar.set)


       canvas.pack(side="left", fill="both", expand=True)

       scrollbar.pack(side="right", fill="y")


       recent = GradeService.get_recent_grades(50)


       if not recent:

           tk.Label(

               scrollable_frame, text="No grades added yet.",

               bg=COLORS['white'], fg=COLORS['text_light']

           ).pack(pady=20)

       else:

           for subject, score, total in recent:

               item = tk.Frame(scrollable_frame, bg=COLORS['white'])

               item.pack(fill='x', pady=5)


               tk.Label(

                   item, text=subject,

                   font=('Segoe UI', 10, 'bold'),

                   bg=COLORS['white'], fg=COLORS['text']

               ).pack(side='left')


               percentage = int((score / total) * 100) if total > 0 else 0

               color = COLORS['success'] if percentage >= 75 else COLORS['danger']


               tk.Label(

                   item, text=f"{int(score)}/{int(total)} ({percentage}%)",

                   font=('Segoe UI', 10, 'bold'),

                   bg=COLORS['white'], fg=color

               ).pack(side='right')


   def create_subjects(self):

       # Subjects management page

       tk.Label(

           self.content, text="Manage Subjects",

           font=('Segoe UI', 18, 'bold'),

           bg=COLORS['bg'], fg=COLORS['text']

       ).pack(anchor='w', pady=(0, 20))


       ModernButton(

           self.content, "+ Add Subject",

           command=self.show_add_subject_dialog

       ).pack(pady=(0, 20))


       container = tk.Frame(self.content, bg=COLORS['bg'])

       container.pack(fill='both', expand=True)


       canvas = tk.Canvas(container, bg=COLORS['bg'], highlightthickness=0)

       scrollbar = tk.Scrollbar(container, orient="vertical", command=canvas.yview)

       scrollable_frame = tk.Frame(canvas, bg=COLORS['bg'])


       scrollable_frame.bind(

           "<Configure>",

           lambda e: canvas.configure(scrollregion=canvas.bbox("all"))

       )


       canvas.create_window((0, 0), window=scrollable_frame, anchor="nw", width=950)

       canvas.configure(yscrollcommand=scrollbar.set)


       canvas.pack(side="left", fill="both", expand=True)

       scrollbar.pack(side="right", fill="y")


       subjects = SubjectService.get_all_subjects()

       for subject_id, name, weights in subjects:

           card = Card(scrollable_frame)

           card.pack(fill='x', pady=5, padx=5)


           content = tk.Frame(card, bg=COLORS['white'])

           content.pack(fill='x', padx=20, pady=15)


           info_frame = tk.Frame(content, bg=COLORS['white'])

           info_frame.pack(side='left', fill='x', expand=True)


           tk.Label(

               info_frame, text=name, font=('Segoe UI', 12, 'bold'),

               bg=COLORS['white'], fg=COLORS['text']

           ).pack(anchor='w')


           if weights and weights.strip():

               weight_text = f"Weights: {weights}"

           else:

               weight_text = "Weights: No weight configuration (Simple Average)"


           tk.Label(

               info_frame, text=weight_text,

               font=('Segoe UI', 9), bg=COLORS['white'],

               fg=COLORS['text_light'], wraplength=600, justify='left'

           ).pack(anchor='w', pady=(5, 0))


           tk.Button(

               content, text="Edit", font=('Segoe UI', 9),

               bg=COLORS['accent'], fg=COLORS['primary'],

               relief='flat', padx=15, pady=5, cursor='hand2',

               command=lambda s=subject_id, n=name, w=weights:

               self.show_edit_subject_dialog(s, n, w)

           ).pack(side='right')


   def create_grades(self):

       # Grades management page

       tk.Label(

           self.content,

           text="Grade Records",

           font=('Segoe UI', 18, 'bold'),

           relief='flat',

           borderwidth=0,

           highlightthickness=0,

           bg=COLORS['bg'], fg=COLORS['text']

       ).pack(anchor='w', pady=(0, 20))


       # Controls

       controls = tk.Frame(self.content, bg=COLORS['bg'])

       controls.pack(fill='x', pady=(0, 20))


       self.grade_filter = tk.StringVar(value="All Subjects")

       filter_combo = ttk.Combobox(

           controls, textvariable=self.grade_filter,

           values=["All Subjects"] + SubjectService.get_subject_names(),

           state="readonly", width=20

       )

       filter_combo.pack(side='left', padx=(0, 10))

       filter_combo.bind("<<ComboboxSelected>>", lambda e: self.refresh_grades())


       ModernButton(

           controls, "+ Add Grade",

           command=self.show_add_grade_dialog, style='success'

       ).pack(side='left')


       # Table

       table_container = tk.Frame(

           self.content,

           bg=COLORS['bg'],

           borderwidth=0,

           highlightthickness=0

       )

       table_container.pack(fill='both', expand=True)


       style = ttk.Style()

       style.theme_use('clam')

       style.configure("Treeview",

                       borderwidth=0,

                       highlightthickness=0,

                       relief='flat',

                       background="white",

                       fieldbackground="white")


       style.layout("Treeview", [('Treeview.treearea', {'sticky': 'nswe'})])


       scrollbar = ttk.Scrollbar(table_container, orient="vertical")

       scrollbar.pack(side='right', fill='y')


       self.grades_tree = ttk.Treeview(

           table_container,

           columns=('Subject', 'Type', 'Score', 'Grade'),

           show='headings',

           height=15,

           yscrollcommand=scrollbar.set,

           style="Treeview"

       )


       scrollbar.config(command=self.grades_tree.yview)


       for col, width in [('Subject', 200), ('Type', 150), ('Score', 120), ('Grade', 100)]:

           self.grades_tree.heading(col, text=col)

           self.grades_tree.column(col, width=width)


       self.grades_tree.pack(side='left', fill='both', expand=True)

       self.grades_tree.bind('<Double-1>', self.delete_grade)


       self.refresh_grades()


   def create_analytics(self):

       # Analytics overview page

       tk.Label(

           self.content, text="Performance Analytics",

           font=('Segoe UI', 18, 'bold'),

           bg=COLORS['bg'], fg=COLORS['text']

       ).pack(anchor='w', pady=(0, 20))


       info_text = """

       This section provides comprehensive analytics about your academic performance:


       📊 Heat Map - Visual representation of pass/fail status for each subject

       👁 Transparency - Detailed breakdown of grading formulas and calculations

       📈 Progress Tracking - Track your improvement over time

       """


       card = Card(self.content)

       card.pack(fill='both', expand=True, pady=10)


       tk.Label(

           card, text=info_text, font=('Segoe UI', 11),

           bg=COLORS['white'], fg=COLORS['text'],

           justify='left'

       ).pack(padx=40, pady=40)


       btn_frame = tk.Frame(card, bg=COLORS['white'])

       btn_frame.pack(pady=20)


       ModernButton(

           btn_frame, "View Heat Map",

           command=lambda: self.show_page('grade_heat_map')

       ).pack(side='left', padx=10)


       ModernButton(

           btn_frame, "View Transparency",

           command=lambda: self.show_page('transparency')

       ).pack(side='left', padx=10)


       ModernButton(

           btn_frame, "View Progress",

           command=lambda: self.show_page('progress')

       ).pack(side='left', padx=10)


   def create_grade_heat_map(self):

       # Grade heat map

       # Header

       tk.Label(

           self.content, text="Subject Status Heat Map",

           font=('Segoe UI', 18, 'bold'),

           bg=COLORS['bg'], fg=COLORS['text']

       ).pack(anchor='w', pady=(0, 20))


       # Legend

       legend_frame = tk.Frame(self.content, bg=COLORS['bg'])

       legend_frame.pack(fill='x', pady=(0, 10))


       tk.Label(

           legend_frame, text="🟢 Passing (≥75%)",

           font=('Segoe UI', 10), bg=COLORS['bg'], fg=COLORS['success']

       ).pack(side='left', padx=10)


       tk.Label(

           legend_frame, text="🔴 Failing (<75%)",

           font=('Segoe UI', 10), bg=COLORS['bg'], fg=COLORS['danger']

       ).pack(side='left', padx=10)


       # Main Container Card

       card = Card(self.content)

       card.pack(fill='both', expand=True, padx=5, pady=5)


       # --- CANVAS & SCROLLBAR SETUP ---

       container = tk.Frame(card, bg=COLORS['white'])

       container.pack(fill='both', expand=True)


       # We add a fixed width/height to ensure it initializes with space

       canvas = tk.Canvas(container, bg=COLORS['white'], highlightthickness=0)

       scrollbar = tk.Scrollbar(container, orient="vertical", command=canvas.yview)


       canvas.pack(side="left", fill="both", expand=True)

       scrollbar.pack(side="right", fill="y")


       canvas.configure(yscrollcommand=scrollbar.set)

       


       subjects = SubjectService.get_all_subjects()


       if not subjects:

           canvas.create_text(

               200, 100, text="No subjects available",

               font=('Segoe UI', 14), fill=COLORS['text_light']

           )

           return


       canvas.delete("all")


       cols = 4

       cell_width = 180

       cell_height = 100

       padding = 20  #


       for idx, (sid, name, weights) in enumerate(subjects):

           

           raw_avg = StatisticsService.compute_weighted_grade(name, weights)

           avg = float(raw_avg) if raw_avg is not None else 0.0


           color = COLORS['success'] if avg >= 75 else COLORS['danger']

           status = "PASSING" if avg >= 75 else "FAILING"


           row = idx // cols

           col = idx % cols


           # Calculate coordinates

           x1 = col * (cell_width + padding) + padding

           y1 = row * (cell_height + padding) + padding

           x2 = x1 + cell_width

           y2 = y1 + cell_height


           # Draw Rectangle

           canvas.create_rectangle(

               x1, y1, x2, y2,

               fill=color, outline='#EEE', width=1

           )


           # Subject Name (Centered in box)

           canvas.create_text(

               x1 + cell_width / 2, y1 + 25,

               text=name[:20], fill='white',

               font=('Segoe UI', 10, 'bold'),

               width=cell_width - 20, justify='center'

           )


           # Grade Percentage

           canvas.create_text(

               x1 + cell_width / 2, y1 + 55,

               text=f"{avg:.1f}%", fill='white',

               font=('Segoe UI', 16, 'bold')

           )


           # Status Label

           canvas.create_text(

               x1 + cell_width / 2, y1 + 82,

               text=status, fill='white',

               font=('Segoe UI', 8, 'bold')

           )


       

       canvas.update_idletasks()

       

       canvas.config(scrollregion=canvas.bbox("all"))


       def _on_mousewheel(event):

           canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")


       canvas.bind_all("<MouseWheel>", _on_mousewheel)


   def create_transparency(self):

       # Transparency Mode

       tk.Label(

           self.content, text="Grade Calculation Transparency",

           font=('Segoe UI', 18, 'bold'),

           bg=COLORS['bg'], fg=COLORS['text']

       ).pack(anchor='w', pady=(0, 20))


       selector_frame = tk.Frame(self.content, bg=COLORS['bg'])

       selector_frame.pack(fill='x', pady=(0, 20))


       tk.Label(

           selector_frame, text="Select Subject:",

           font=('Segoe UI', 11), bg=COLORS['bg']

       ).pack(side='left', padx=(0, 10))


       self.transparency_subject = tk.StringVar()

       subjects = SubjectService.get_subject_names()


       if subjects:

           self.transparency_subject.set(subjects[0])


       subject_combo = ttk.Combobox(

           selector_frame, textvariable=self.transparency_subject,

           values=subjects, state='readonly', width=30

       )

       subject_combo.pack(side='left')

       subject_combo.bind("<<ComboboxSelected>>", lambda e: self.refresh_transparency())


       container = tk.Frame(self.content, bg=COLORS['bg'])

       container.pack(fill='both', expand=True)


       canvas = tk.Canvas(container, bg=COLORS['bg'], highlightthickness=0)

       scrollbar = tk.Scrollbar(container, orient="vertical", command=canvas.yview)

       self.transparency_frame = tk.Frame(canvas, bg=COLORS['bg'])


       self.transparency_frame.bind(

           "<Configure>",

           lambda e: canvas.configure(scrollregion=canvas.bbox("all"))

       )


       canvas.create_window((0, 0), window=self.transparency_frame, anchor="nw", width=950)

       canvas.configure(yscrollcommand=scrollbar.set)


       canvas.pack(side="left", fill="both", expand=True)

       scrollbar.pack(side="right", fill="y")


       self.refresh_transparency()


   def refresh_transparency(self):

       # Refresh transparency view

       for widget in self.transparency_frame.winfo_children():

           widget.destroy()


       subject_name = self.transparency_subject.get()

       if not subject_name:

           return


       subjects = SubjectService.get_all_subjects()

       weights_str = ""

       for sid, name, weights in subjects:

           if name == subject_name:

               weights_str = weights

               break


       formula_card = Card(self.transparency_frame)

       formula_card.pack(fill='x', pady=10, padx=5)


       tk.Label(

           formula_card, text="📝 Grading Formula",

           font=('Segoe UI', 14, 'bold'),

           bg=COLORS['white'], fg=COLORS['text']

       ).pack(anchor='w', padx=20, pady=(15, 10))


       if weights_str:

           formula_text = f"Weighted Average: {weights_str}"

           tk.Label(

               formula_card, text=formula_text,

               font=('Segoe UI', 11),

               bg=COLORS['white'], fg=COLORS['text']

           ).pack(anchor='w', padx=20, pady=(0, 15))

       else:

           tk.Label(

               formula_card, text="Simple Average: (Sum of Scores / Sum of Total) × 100",

               font=('Segoe UI', 11),

               bg=COLORS['white'], fg=COLORS['text']

           ).pack(anchor='w', padx=20, pady=(0, 15))


       breakdown = StatisticsService.get_formula_breakdown(subject_name, weights_str)


       if breakdown:

           breakdown_card = Card(self.transparency_frame)

           breakdown_card.pack(fill='x', pady=10, padx=5)


           tk.Label(

               breakdown_card, text="📊 Category Breakdown",

               font=('Segoe UI', 14, 'bold'),

               bg=COLORS['white'], fg=COLORS['text']

           ).pack(anchor='w', padx=20, pady=(15, 10))


           for item in breakdown:

               item_frame = tk.Frame(breakdown_card, bg=COLORS['white'])

               item_frame.pack(fill='x', padx=20, pady=5)


               tk.Label(

                   item_frame, text=item['category'],

                   font=('Segoe UI', 11, 'bold'),

                   bg=COLORS['white'], width=15, anchor='w'

               ).pack(side='left')


               tk.Label(

                   item_frame, text=f"Average: {item['average']}%",

                   font=('Segoe UI', 10),

                   bg=COLORS['white'], width=20, anchor='w'

               ).pack(side='left')


               tk.Label(

                   item_frame, text=f"Weight: {item['weight_percentage']}",

                   font=('Segoe UI', 10),

                   bg=COLORS['white'], width=15, anchor='w'

               ).pack(side='left')


               tk.Label(

                   item_frame, text=f"Contribution: {item['contribution']}%",

                   font=('Segoe UI', 10, 'bold'),

                   bg=COLORS['white'], fg=COLORS['primary']

               ).pack(side='left')


           tk.Frame(breakdown_card, bg=COLORS['white'], height=15).pack()


       final_grade = StatisticsService.compute_weighted_grade(subject_name, weights_str)


       final_card = Card(self.transparency_frame)

       final_card.pack(fill='x', pady=10, padx=5)


       tk.Label(

           final_card, text="🎯 Final Grade",

           font=('Segoe UI', 14, 'bold'),

           bg=COLORS['white'], fg=COLORS['text']

       ).pack(anchor='w', padx=20, pady=(15, 10))


       grade_color = COLORS['success'] if final_grade >= 75 else COLORS['danger']


       tk.Label(

           final_card, text=f"{final_grade:.2f}%",

           font=('Segoe UI', 24, 'bold'),

           bg=COLORS['white'], fg=grade_color

       ).pack(pady=(0, 15))


   def create_progress_tracking(self):

       # Progress tracking

       tk.Label(

           self.content, text="Progress Tracking",

           font=('Segoe UI', 18, 'bold'),

           bg=COLORS['bg'], fg=COLORS['text']

       ).pack(anchor='w', pady=(0, 20))


       selector_frame = tk.Frame(self.content, bg=COLORS['bg'])

       selector_frame.pack(fill='x', pady=(0, 20))


       tk.Label(

           selector_frame, text="Select Subject:",

           font=('Segoe UI', 11), bg=COLORS['bg']

       ).pack(side='left', padx=(0, 10))


       self.progress_subject = tk.StringVar(value="All Subjects")

       subjects = ["All Subjects"] + SubjectService.get_subject_names()


       subject_combo = ttk.Combobox(

           selector_frame, textvariable=self.progress_subject,

           values=subjects, state='readonly', width=30

       )

       subject_combo.pack(side='left')

       subject_combo.bind("<<ComboboxSelected>>", lambda e: self.refresh_progress())


       self.progress_container = tk.Frame(self.content, bg=COLORS['bg'])

       self.progress_container.pack(fill='both', expand=True)


       self.refresh_progress()


   def refresh_progress(self):

       # Refresh progress view

       for widget in self.progress_container.winfo_children():

           widget.destroy()


       subject_name = self.progress_subject.get()

       if subject_name == "All Subjects":

           subject_name = None


       trend = StatisticsService.get_progress_trend(subject_name)


       if not trend:

           tk.Label(

               self.progress_container, text="No data available for progress tracking",

               font=('Segoe UI', 12), bg=COLORS['bg'], fg=COLORS['text_light']

           ).pack(pady=50)

           return


       card = Card(self.progress_container)

       card.pack(fill='both', expand=True, pady=10)


       canvas = tk.Canvas(card, bg=COLORS['white'], height=400, highlightthickness=0)

       canvas.pack(fill='both', expand=True, padx=20, pady=20)


       width = 900

       height = 350

       margin = 50


       canvas.create_line(margin, height - margin, width - margin, height - margin, width=2)

       canvas.create_line(margin, margin, margin, height - margin, width=2)


       canvas.create_text(width / 2, height - 10, text="Timeline", font=('Segoe UI', 10, 'bold'))

       canvas.create_text(20, height / 2, text="Average (%)", font=('Segoe UI', 10, 'bold'), angle=90)


       if len(trend) > 1:

           max_avg = max(item['average'] for item in trend)

           min_avg = min(item['average'] for item in trend)

           avg_range = max_avg - min_avg if max_avg > min_avg else 10


           for i in range(5):

               y = margin + i * (height - 2 * margin) / 4

               canvas.create_line(margin, y, width - margin, y, fill='#E5E7EB', dash=(2, 2))

               value = 100 - i * 25

               canvas.create_text(margin - 10, y, text=str(value), font=('Segoe UI', 8))


           points = []

           for i, item in enumerate(trend):

               x = margin + (i / (len(trend) - 1)) * (width - 2 * margin)

               normalized = (item['average'] - min_avg) / avg_range if avg_range > 0 else 0.5

               y = height - margin - normalized * (height - 2 * margin)

               points.append((x, y))


               color = COLORS['success'] if item['average'] >= 75 else COLORS['danger']

               canvas.create_oval(x - 4, y - 4, x + 4, y + 4, fill=color, outline=color)


           for i in range(len(points) - 1):

               canvas.create_line(

                   points[i][0], points[i][1],

                   points[i + 1][0], points[i + 1][1],

                   fill=COLORS['primary'], width=2

               )


           indices = [0, len(trend) // 2, len(trend) - 1]

           for idx in indices:

               if idx < len(trend):

                   x = margin + (idx / (len(trend) - 1)) * (width - 2 * margin)

                   canvas.create_text(x, height - margin + 20, text=trend[idx]['date'],

                                      font=('Segoe UI', 8), angle=45)


       stats_frame = tk.Frame(card, bg=COLORS['white'])

       stats_frame.pack(fill='x', padx=40, pady=20)


       first_avg = trend[0]['average']

       last_avg = trend[-1]['average']

       improvement = last_avg - first_avg


       improvement_color = COLORS['success'] if improvement >= 0 else COLORS['danger']

       improvement_text = f"+{improvement:.2f}%" if improvement >= 0 else f"{improvement:.2f}%"


       tk.Label(

           stats_frame, text="📊 Summary:",

           font=('Segoe UI', 12, 'bold'),

           bg=COLORS['white']

       ).pack(side='left', padx=(0, 20))


       tk.Label(

           stats_frame, text=f"First: {first_avg:.2f}%",

           font=('Segoe UI', 11),

           bg=COLORS['white']

       ).pack(side='left', padx=10)


       tk.Label(

           stats_frame, text=f"Current: {last_avg:.2f}%",

           font=('Segoe UI', 11),

           bg=COLORS['white']

       ).pack(side='left', padx=10)


       tk.Label(

           stats_frame, text=f"Change: {improvement_text}",

           font=('Segoe UI', 11, 'bold'),

           bg=COLORS['white'], fg=improvement_color

       ).pack(side='left', padx=10)


   def create_attachments(self):

       # Attachments management page

       tk.Label(

           self.content, text="File Attachments",

           font=('Segoe UI', 18, 'bold'),

           bg=COLORS['bg'], fg=COLORS['text']

       ).pack(anchor='w', pady=(0, 20))


       ModernButton(

           self.content, "+ Add Attachment",

           command=self.show_add_attachment_dialog, style='success'

       ).pack(pady=(0, 20))


       table_container = tk.Frame(self.content, bg=COLORS['bg'])

       table_container.pack(fill='both', expand=True)


       scrollbar = ttk.Scrollbar(table_container, orient="vertical")

       scrollbar.pack(side='right', fill='y')


       self.attachments_tree = ttk.Treeview(

           table_container,

           columns=('Subject', 'Type', 'File', 'Date'),

           show='headings',

           height=15,

           yscrollcommand=scrollbar.set

       )


       scrollbar.config(command=self.attachments_tree.yview)


       for col, width in [('Subject', 200), ('Type', 150), ('File', 250), ('Date', 150)]:

           self.attachments_tree.heading(col, text=col)

           self.attachments_tree.column(col, width=width)


       self.attachments_tree.pack(side='left', fill='both', expand=True)


       btn_frame = tk.Frame(self.content, bg=COLORS['bg'])

       btn_frame.pack(pady=10)


       ModernButton(

           btn_frame, "Open File",

           command=self.open_attachment

       ).pack(side='left', padx=5)


       ModernButton(

           btn_frame, "Delete",

           command=self.delete_attachment, style='danger'

       ).pack(side='left', padx=5)


       self.refresh_attachments()


   def refresh_attachments(self):

       # Refresh attachments list

       for item in self.attachments_tree.get_children():

           self.attachments_tree.delete(item)


       attachments = AttachmentService.get_all_attachments()

       for att in attachments:

           self.attachments_tree.insert(

               '', 'end', iid=att['id'],

               values=(att['subject'], att['activity_type'],

                       att['file_name'], att['uploaded_at'].split()[0])

           )


   def open_attachment(self):

       # Open selected attachment

       selection = self.attachments_tree.selection()

       if not selection:

           messagebox.showwarning("No Selection", "Please select a file to open")

           return


       attachments = AttachmentService.get_all_attachments()

       for att in attachments:

           if str(att['id']) == selection[0]:

               file_path = att['file_path']

               if os.path.exists(file_path):

                   try:

                       if platform.system() == 'Windows':

                           os.startfile(file_path)

                       elif platform.system() == 'Darwin':

                           subprocess.call(['open', file_path])

                       else:

                           subprocess.call(['xdg-open', file_path])

                   except Exception as e:

                       messagebox.showerror("Error", f"Could not open file: {e}")

               else:

                   messagebox.showerror("Error", "File not found")

               break


   def delete_attachment(self):

       # Delete selected attachment

       selection = self.attachments_tree.selection()

       if not selection:

           messagebox.showwarning("No Selection", "Please select a file to delete")

           return


       if messagebox.askyesno("Confirm Delete", "Delete this attachment?"):

           attachments = AttachmentService.get_all_attachments()

           for att in attachments:

               if str(att['id']) == selection[0]:

                   AttachmentService.delete_attachment(att['id'], att['file_path'])

                   self.refresh_attachments()

                   break


   def show_profile_dialog(self):

       dialog = tk.Toplevel(self.root)

       dialog.title("Edit Profile")

       dialog.geometry("400x350")

       dialog.configure(bg=COLORS['white'])


       frame = tk.Frame(dialog, bg=COLORS['white'], padx=30, pady=30)

       frame.pack(fill='both', expand=True)


       student = StudentService.get_student_profile()


       sid_entry = ModernEntry(frame, "Student ID")

       sid_entry.pack(fill='x', pady=(0, 15))

       if student and student[0] != '---':

           sid_entry.insert(0, student[0])

       elif self.user_data['student_id']:

           sid_entry.insert(0, self.user_data['student_id'])


       course_entry = ModernEntry(frame, "Course")

       course_entry.pack(fill='x', pady=(0, 20))

       if student and student[1] != '---':

           course_entry.insert(0, student[1])

       elif self.user_data['course']:

           course_entry.insert(0, self.user_data['course'])


       def save():

           StudentService.save_student_profile(sid_entry.get(), course_entry.get())

           dialog.destroy()

           self.show_page('dashboard')


       def clear():

           if messagebox.askyesno("Clear Profile", "Clear all profile information?"):

               StudentService.save_student_profile('---', '---')

               dialog.destroy()

               self.show_page('dashboard')


       ModernButton(frame, "Save Profile", command=save, style='success').pack(fill='x', pady=(0, 10))

       ModernButton(frame, "Clear Profile", command=clear, style='danger').pack(fill='x')


   def show_add_subject_dialog(self):

       dialog = tk.Toplevel(self.root)

       dialog.title("Add Subject")

       dialog.geometry("450x450")

       dialog.configure(bg=COLORS['white'])


       frame = tk.Frame(dialog, bg=COLORS['white'], padx=30, pady=30)

       frame.pack(fill='both', expand=True)


       name_entry = ModernEntry(frame, "Subject Name")

       name_entry.pack(fill='x', pady=(0, 15))


       tk.Label(

           frame, text="Weight Configuration (Optional)",

           bg=COLORS['white'], fg=COLORS['text'],

           font=('Segoe UI', 9, 'bold')

       ).pack(anchor='w', pady=(0, 5))


       tk.Label(

           frame, text="Format: ActivityType:Weight, ActivityType:Weight",

           bg=COLORS['white'], fg=COLORS['text_light'],

           font=('Segoe UI', 8)

       ).pack(anchor='w')


       tk.Label(

           frame, text="Example: Quiz:30, Exam:40, Project:30",

           bg=COLORS['white'], fg=COLORS['text_light'],

           font=('Segoe UI', 8)

       ).pack(anchor='w', pady=(0, 5))


       weights_text = tk.Text(frame, height=4, font=('Segoe UI', 9),

                              relief='flat', borderwidth=0)

       weights_text.pack(fill='x', pady=(5, 20))


       def save():

           name = name_entry.get().strip()

           weights = weights_text.get("1.0", "end-1c").strip()


           if not name:

               messagebox.showerror("Error", "Subject name is required")

               return


           SubjectService.add_subject(name, weights)

           dialog.destroy()

           self.show_page('subjects')


       def cancel():

           dialog.destroy()


       ModernButton(frame, "Add Subject", command=save, style='success').pack(fill='x', pady=(0, 10))

       ModernButton(frame, "Cancel", command=cancel, style='secondary').pack(fill='x')


   def show_edit_subject_dialog(self, sid, name, weights):

       dialog = tk.Toplevel(self.root)

       dialog.title("Edit Subject")

       dialog.geometry("450x500")

       dialog.configure(bg=COLORS['white'])


       frame = tk.Frame(dialog, bg=COLORS['white'], padx=30, pady=30)

       frame.pack(fill='both', expand=True)


       name_entry = ModernEntry(frame, "Subject Name")

       name_entry.insert(0, name)

       name_entry.pack(fill='x', pady=(0, 15))


       tk.Label(

           frame, text="Weight Configuration (Optional)",

           bg=COLORS['white'], fg=COLORS['text'],

           font=('Segoe UI', 9, 'bold')

       ).pack(anchor='w', pady=(0, 5))


       tk.Label(

           frame, text="Format: ActivityType:Weight, ActivityType:Weight",

           bg=COLORS['white'], fg=COLORS['text_light'],

           font=('Segoe UI', 8)

       ).pack(anchor='w')


       tk.Label(

           frame, text="Example: Quiz:30, Exam:40, Project:30",

           bg=COLORS['white'], fg=COLORS['text_light'],

           font=('Segoe UI', 8)

       ).pack(anchor='w', pady=(0, 5))


       weights_text = tk.Text(frame, height=4, font=('Segoe UI', 9),

                              relief='flat', borderwidth=0)

       weights_text.insert("1.0", weights if weights else "")

       weights_text.pack(fill='x', pady=(5, 20))


       def save():

           new_name = name_entry.get().strip()

           new_weights = weights_text.get("1.0", "end-1c").strip()


           if new_name:

               SubjectService.update_subject(sid, new_name, new_weights, name)

               dialog.destroy()

               self.show_page('subjects')

           else:

               messagebox.showerror("Error", "Subject name is required")


       def delete():

           if messagebox.askyesno("Delete", f"Delete '{name}'? All grades will be removed."):

               SubjectService.delete_subject(sid, name)

               dialog.destroy()

               self.show_page('subjects')


       ModernButton(frame, "Save Changes", command=save, style='success').pack(fill='x', pady=(0, 10))

       ModernButton(frame, "Delete Subject", command=delete, style='danger').pack(fill='x')


   def show_add_grade_dialog(self):

      # Add Grade Dialog

       dialog = tk.Toplevel(self.root)

       dialog.title("Add Grade")

       dialog.geometry("450x500")

       dialog.configure(bg=COLORS['white'])


       frame = tk.Frame(

           dialog,

           bg=COLORS['white'],

           padx=30,

           pady=30,

           borderwidth=0,

           highlightthickness=0,

           relief='flat'

       )

       frame.pack(fill='both', expand=True)


       subjects = SubjectService.get_subject_names()


       if not subjects:

           tk.Label(

               frame, text="❌ No subjects found!",

               fg=COLORS['danger'], bg=COLORS['white'],

               font=('Segoe UI', 12, 'bold')

           ).pack(pady=10)

           tk.Label(

               frame, text="Please add at least one subject first.\nGo to Subjects page.",

               bg=COLORS['white'], fg=COLORS['text_light']

           ).pack(pady=10)

           ModernButton(frame, "Close", command=dialog.destroy).pack(fill='x', pady=10)

           return


       tk.Label(

           frame, text="Subject",

           bg=COLORS['white'], fg=COLORS['text'],

           font=('Segoe UI', 10, 'bold')

       ).pack(anchor='w', pady=(0, 5))


       subject_var = tk.StringVar(value=subjects[0])

       subject_combo = ttk.Combobox(

           frame, textvariable=subject_var,

           values=subjects, state='readonly', width=40

       )

       subject_combo.pack(fill='x', pady=(0, 15))


       activity_entry = ModernEntry(frame, "Activity Type (e.g. Quiz, Exam, Midterm)")

       activity_entry.pack(fill='x', pady=(0, 15))


       score_entry = ModernEntry(frame, "Score")

       score_entry.pack(fill='x', pady=(0, 15))


       total_entry = ModernEntry(frame, "Total Score")

       total_entry.pack(fill='x', pady=(0, 20))


       def save_grade():

           subject = subject_var.get().strip()

           activity = activity_entry.get().strip()

           score_str = score_entry.get().strip()

           total_str = total_entry.get().strip()


           if not subject:

               messagebox.showerror("Error", "Subject not selected!")

               return


           if not activity:

               messagebox.showerror("Error", "Please enter activity type!")

               return


           if not score_str:

               messagebox.showerror("Error", "Please enter score!")

               return


           if not total_str:

               messagebox.showerror("Error", "Please enter total score!")

               return


           try:

               score = float(score_str)

               total = float(total_str)

           except ValueError:

               messagebox.showerror("Error", "Scores must be numbers!\nExample: 85 or 85.5")

               return


           if score < 0 or total < 0:

               messagebox.showerror("Error", "Scores cannot be negative!")

               return


           if total == 0:

               messagebox.showerror("Error", "Total score must be greater than 0!")

               return


           if score > total:

               messagebox.showerror("Error", f"Score ({score}) cannot be more than total ({total})!")

               return


           try:

               GradeService.add_grade(subject, activity, score, total)

               messagebox.showinfo("Success", "✅ Grade added successfully!")

               dialog.destroy()

               self.refresh_grades()

               self.show_page('grades')

           except Exception as e:

               messagebox.showerror("Database Error", f"Failed to save grade:\n{str(e)}")


       button_frame = tk.Frame(frame, bg=COLORS['white'])

       button_frame.pack(fill='x', pady=(20, 0))


       ModernButton(button_frame, "Add Grade", command=save_grade, style='success').pack(side='left', fill='x', expand=True, padx=(0, 5))

       ModernButton(button_frame, "Cancel", command=dialog.destroy, style='secondary').pack(side='left', fill='x', expand=True, padx=(5, 0))


   def show_add_attachment_dialog(self):

     

       dialog = tk.Toplevel(self.root)

       dialog.title("Add Attachment")

       dialog.geometry("400x400")

       dialog.configure(bg=COLORS['white'])


       frame = tk.Frame(dialog, bg=COLORS['white'], padx=30, pady=30)

       frame.pack(fill='both', expand=True)


       tk.Label(frame, text="Subject", bg=COLORS['white'],

                font=('Segoe UI', 9, 'bold')).pack(anchor='w')

       subject_var = tk.StringVar()

       subjects = SubjectService.get_subject_names()


       if not subjects:

           tk.Label(frame, text="Please add a subject first!",

                    fg='red', bg=COLORS['white']).pack()

           return


       subject_combo = ttk.Combobox(frame, textvariable=subject_var,

                                    values=subjects, state='readonly')

       subject_combo.pack(fill='x', pady=(5, 15))


       type_entry = ModernEntry(frame, "Activity Type (e.g. Quiz, Exam, Project)")

       type_entry.pack(fill='x', pady=(0, 15))


       tk.Label(frame, text="File", bg=COLORS['white'],

                font=('Segoe UI', 9, 'bold')).pack(anchor='w', pady=(0, 5))


       file_frame = tk.Frame(frame, bg=COLORS['white'])

       file_frame.pack(fill='x', pady=(0, 20))


       file_path_var = tk.StringVar(value="No file selected")

       tk.Label(

           file_frame, textvariable=file_path_var,

           bg=COLORS['white'], fg=COLORS['text_light'],

           font=('Segoe UI', 9)

       ).pack(side='left', fill='x', expand=True)


       def browse_file():

           filename = filedialog.askopenfilename(

               title="Select File",

               filetypes=[

                   ("All Files", "*.*"),

                   ("PDF Files", "*.pdf"),

                   ("Image Files", "*.png *.jpg *.jpeg"),

                   ("Document Files", "*.docx *.doc *.txt")

               ]

           )

           if filename:

               file_path_var.set(os.path.basename(filename))

               browse_file.selected_path = filename


       browse_file.selected_path = None


       tk.Button(

           file_frame, text="Browse", font=('Segoe UI', 9),

           bg=COLORS['accent'], fg=COLORS['primary'],

           relief='flat', padx=15, pady=5, cursor='hand2',

           command=browse_file

       ).pack(side='right', padx=(10, 0))


       def save():

           if not browse_file.selected_path:

               messagebox.showerror("Error", "Please select a file")

               return


           subject = subject_var.get()

           activity_type = type_entry.get().strip()


           if not subject or not activity_type:

               messagebox.showerror("Error", "Please fill all fields")

               return


           success = AttachmentService.save_attachment(

               subject, activity_type, browse_file.selected_path

           )


           if success:

               dialog.destroy()

               self.show_page('attachments')

           else:

               messagebox.showerror("Error", "Failed to save attachment")


       ModernButton(frame, "Add Attachment", command=save, style='success').pack(fill='x')


   def refresh_grades(self):

       for item in self.grades_tree.get_children():

           self.grades_tree.delete(item)


       filter_val = self.grade_filter.get()

       filter_val = None if filter_val == "All Subjects" else filter_val


       for gid, subj, atype, score, total in GradeService.get_all_grades(filter_val):

           pct = f"{int((score / total) * 100)}%" if total > 0 else "0%"

           self.grades_tree.insert('', 'end', iid=gid,

                                   values=(subj, atype, f"{int(score)}/{int(total)}", pct))


   def delete_grade(self, event):

       item = self.grades_tree.selection()

       if item and messagebox.askyesno("Delete", "Remove this grade?"):

           GradeService.delete_grade(int(item[0]))

           self.refresh_grades()


def resource_path(relative_path):

   # for logo

   try:

       base_path = sys._MEIPASS

   except Exception:

       base_path = os.path.abspath(".")


   return os.path.join(base_path, relative_path)


logo_path = resource_path("LOGO.png")


if __name__ == "__main__":

   from engine2 import setup_database


   setup_database()

   root = tk.Tk()

   root.attributes('-fullscreen', True)

   root.bind("<Escape>", lambda event: root.attributes("-fullscreen", False))


   AuthService.setup_auth_database()


   def on_login_success(user_data):

       app = GradeTrackerApp(root, user_data)


   show_login(root, on_login_success)

   root.mainloop()



LOGIN


import tkinter as tk

from tkinter import messagebox, ttk

import sqlite3

import hashlib

from datetime import datetime

from PIL import Image, ImageTk

import os

import sys


# Color Pallete

COLORS = {

   'primary': '#DC143C',

   'primary_dark': '#B91C1C',

   'accent': '#FEE2E2',

   'white': '#FFFFFF',

   'bg': '#F9FAFB',

   'text': '#1F2937',

   'text_light': '#6B7280',

   'success': '#10B981',

   'danger': '#EF4444',

}


DATABASE = 'tracker.db'



class CredentialManager:

   # Remember me


   @staticmethod

   def setup_credentials_table():

     

       conn = sqlite3.connect(DATABASE)

       cursor = conn.cursor()


       cursor.execute('''

           CREATE TABLE IF NOT EXISTS saved_credentials

           (

               id INTEGER PRIMARY KEY CHECK (id = 1),

               username TEXT NOT NULL,

               password_hash TEXT NOT NULL,

               timestamp TEXT DEFAULT CURRENT_TIMESTAMP

           )

       ''')


       conn.commit()

       conn.close()


   @staticmethod

   def save_credentials(username, password_hash):

       # Save credentials to tracker.db

       try:

           conn = sqlite3.connect(DATABASE)

           cursor = conn.cursor()


           cursor.execute('DELETE FROM saved_credentials')


           cursor.execute('''

               INSERT INTO saved_credentials (id, username, password_hash, timestamp)

               VALUES (1, ?, ?, ?)

           ''', (username, password_hash, datetime.now().strftime("%Y-%m-%d %H:%M:%S")))


           conn.commit()

           conn.close()

           return True

       except Exception as e:

           print(f"Error saving credentials: {e}")

           return False


   @staticmethod

   def load_credentials():

       # Load saved credentials from db

       try:

           conn = sqlite3.connect(DATABASE)

           cursor = conn.cursor()


           cursor.execute('SELECT username, password_hash FROM saved_credentials WHERE id = 1')

           result = cursor.fetchone()


           conn.close()


           if result:

               return result[0], result[1]

           return None, None

       except Exception as e:

           print(f"Error loading credentials: {e}")

           return None, None


   @staticmethod

   def clear_credentials():

       # Clear saved credentials from database

       try:

           conn = sqlite3.connect(DATABASE)

           cursor = conn.cursor()


           cursor.execute('DELETE FROM saved_credentials')


           conn.commit()

           conn.close()

           return True

       except Exception as e:

           print(f"Error clearing credentials: {e}")

           return False



class AuthService:

   @staticmethod

   def hash_password(password):

       return hashlib.sha256(password.encode()).hexdigest()


   @staticmethod

   def setup_auth_database():

       conn = sqlite3.connect(DATABASE)

       cursor = conn.cursor()

       cursor.execute('''CREATE TABLE IF NOT EXISTS users (

                          id INTEGER PRIMARY KEY AUTOINCREMENT,

                          username TEXT UNIQUE NOT NULL,

                          password TEXT NOT NULL,

                          student_id TEXT, course TEXT, full_name TEXT,

                          created_at TEXT DEFAULT CURRENT_TIMESTAMP,

                          last_login TEXT)''')

       cursor.execute('''CREATE TABLE IF NOT EXISTS local_prefs (

                          key TEXT PRIMARY KEY, value TEXT)''')

       conn.commit()

       conn.close()


   @staticmethod

   def register_user(username, password, student_id, course, full_name):

       conn = sqlite3.connect(DATABASE)

       cursor = conn.cursor()

       try:

           hashed_pw = AuthService.hash_password(password)

           cursor.execute("INSERT INTO users (username, password, student_id, course, full_name) VALUES (?, ?, ?, ?, ?)",

                          (username, hashed_pw, student_id, course, full_name))

           conn.commit()

           return True, "Registration successful!"

       except:

           return False, "Username exists!"

       finally:

           conn.close()


   @staticmethod

   def login_user(username, password):

       conn = sqlite3.connect(DATABASE)

       cursor = conn.cursor()

       hashed_pw = AuthService.hash_password(password)

       cursor.execute("SELECT id, student_id, course, full_name FROM users WHERE username = ? AND password = ?",

                      (username, hashed_pw))

       result = cursor.fetchone()

       conn.close()

       if result:

           return True, {'user_id': result[0], 'student_id': result[1], 'course': result[2], 'full_name': result[3],

                         'username': username}

       return False, None


   @staticmethod

   def verify_stored_credentials(username, password_hash):

       # Validate credentials using stored hash

       conn = sqlite3.connect(DATABASE)

       cursor = conn.cursor()


       cursor.execute('''

                      SELECT id, student_id, course, full_name

                      FROM users

                      WHERE username = ?

                        AND password = ?

                      ''', (username, password_hash))


       result = cursor.fetchone()


       if result:

           cursor.execute('''

                          UPDATE users

                          SET last_login = ?

                          WHERE id = ?

                          ''', (datetime.now().strftime("%Y-%m-%d %H:%M:%S"), result[0]))

           conn.commit()


           conn.close()

           return True, {

               'user_id': result[0],

               'student_id': result[1],

               'course': result[2],

               'full_name': result[3],

               'username': username

           }

       else:

           conn.close()

           return False, None


   @staticmethod

   def check_username_exists(username):

       # Check if username already exists

       conn = sqlite3.connect(DATABASE)

       cursor = conn.cursor()


       cursor.execute('SELECT COUNT(*) FROM users WHERE username = ?', (username,))

       count = cursor.fetchone()[0]


       conn.close()

       return count > 0



class ModernEntry(tk.Frame):

   # Modern entry field with label


   def __init__(self, parent, label, show=None, **kwargs):

       super().__init__(parent, bg=COLORS['white'])


       tk.Label(

           self, text=label, bg=COLORS['white'],

           fg=COLORS['text'], font=('Segoe UI', 9, 'bold')

       ).pack(anchor='w', pady=(0, 5))


       self.entry = tk.Entry(

           self,

           font=('Segoe UI', 10),

           relief='flat',

           bg=COLORS['bg'],

           highlightthickness=1,

           highlightbackground="#E5E7EB",

           highlightcolor=COLORS['primary'],

           show=show,

           **kwargs

       )

       self.entry.pack(fill='x', ipady=8)


   def get(self):

       return self.entry.get()


   def insert(self, index, string):

       self.entry.insert(index, string)


   def delete(self, first, last=None):

       self.entry.delete(first, last)


   def focus(self):

       self.entry.focus()



class ModernButton(tk.Button):

   


   def __init__(self, parent, text, command=None, style='primary', **kwargs):

       colors = {

           'primary': (COLORS['primary'], COLORS['white']),

           'secondary': (COLORS['white'], COLORS['primary']),

           'success': (COLORS['success'], COLORS['white']),

       }

       bg, fg = colors.get(style, colors['primary'])


       super().__init__(

           parent, text=text, command=command,

           bg=bg, fg=fg, font=('Segoe UI', 10, 'bold'),

           relief='flat', cursor='hand2',

           padx=20, pady=10, **kwargs

       )



class LoginPage:

   # Login page UI


   def __init__(self, root, on_success):

       self.root = root

       self.on_success = on_success

       self.auto_login_attempted = False


       # Clear window

       for widget in root.winfo_children():

           widget.destroy()


       # Layout Setup

       self.root.grid_columnconfigure(0, weight=1)

       self.root.grid_rowconfigure(0, weight=1)


       # Main background frame

       main_frame = tk.Frame(self.root, bg=COLORS['bg'])

       main_frame.grid(row=0, column=0, sticky="nsew")


       main_frame.grid_columnconfigure(0, weight=1)

       main_frame.grid_rowconfigure(0, weight=1)


       # Login card

       card = tk.Frame(

           main_frame,

           bg=COLORS['white'],

           relief='flat',

           borderwidth=0,

           highlightthickness=0,

       )

       card.grid(row=0, column=0, padx=40, pady=40)


       # Inner content container

       content = tk.Frame(card, bg=COLORS['white'], padx=50, pady=30)

       content.pack(fill='both', expand=True)


       # Logo / Header

       icon_path = resource_path("LOGO.png")


       try:

           img = Image.open(icon_path)

           img = img.resize((80, 80), Image.Resampling.LANCZOS)

           self.grad_icon = ImageTk.PhotoImage(img)

           tk.Label(content, image=self.grad_icon, bg=COLORS['white']).pack(pady=(0, 10))

       except Exception as e:

           print(f"Logo error: {e}")

           tk.Label(content, text="🎓", font=('Segoe UI', 48), bg=COLORS['white']).pack(pady=(0, 10))


       tk.Label(

           content, text="PUP GRADE TRACKER",

           font=('Segoe UI', 20, 'bold'),

           bg=COLORS['white'], fg=COLORS['primary'],

       ).pack()


       tk.Label(

           content, text="Polytechnic University of The Philippines",

           font=('Segoe UI', 9),

           bg=COLORS['white'], fg=COLORS['text_light']

       ).pack(pady=(0, 30))


       # Login form

       self.username_entry = ModernEntry(content, "Username")

       self.username_entry.pack(fill='x', pady=(0, 15))


       self.password_entry = ModernEntry(content, "Password", show="●")

       self.password_entry.pack(fill='x', pady=(0, 25))


       # Remember me checkbox

       remember_frame = tk.Frame(content, bg=COLORS['white'])

       remember_frame.pack(fill='x', pady=(0, 20))


       self.remember_var = tk.BooleanVar()

       tk.Checkbutton(

           remember_frame, text="Remember me",

           variable=self.remember_var,

           bg=COLORS['white'], font=('Segoe UI', 9),

           activebackground=COLORS['white']

       ).pack(side='left')


       # Login button

       ModernButton(

           content, "Login",

           command=self.handle_login,

           style='primary'

       ).pack(fill='x', ipady=5)


       # Register link

       register_frame = tk.Frame(content, bg=COLORS['white'])

       register_frame.pack(pady=(15, 0))


       tk.Label(

           register_frame, text="Don't have an account?",

           bg=COLORS['white'], fg=COLORS['text_light'],

           font=('Segoe UI', 9)

       ).pack(side='left')


       register_btn = tk.Label(

           register_frame, text="Register here",

           bg=COLORS['white'], fg=COLORS['primary'],

           font=('Segoe UI', 9, 'bold'),

           cursor='hand2'

       )

       register_btn.pack(side='left', padx=(5, 0))

       register_btn.bind('<Button-1>', lambda e: self.show_register())

       register_btn.bind('<Enter>', lambda e: register_btn.config(fg=COLORS['primary_dark']))

       register_btn.bind('<Leave>', lambda e: register_btn.config(fg=COLORS['primary']))


       # Footer

       tk.Label(

           content, text="© 2026 PUP Grade Tracker. All rights reserved.",

           font=('Segoe UI', 7),

           bg=COLORS['white'], fg=COLORS['text_light']

       ).pack(pady=(30, 0))


       # Bind Enter key

       self.username_entry.entry.bind('<Return>', lambda e: self.password_entry.focus())

       self.password_entry.entry.bind('<Return>', lambda e: self.handle_login())


       # Load saved credentials if available

       self.load_saved_credentials()


       # Focus appropriate field

       if self.username_entry.get():

           self.password_entry.focus()

       else:

           self.username_entry.focus()


   def load_saved_credentials(self):

       # Load and auto-fill saved credentials

       username, password_hash = CredentialManager.load_credentials()


       if username and password_hash:

           self.username_entry.insert(0, username)

           self.remember_var.set(True)

           self.password_entry.insert(0, "●●●●●●●●")

           self.stored_password_hash = password_hash

           self.password_entry.entry.config(fg=COLORS['text_light'])


   def handle_login(self):

       # Handle login attempt

       username = self.username_entry.get().strip()

       password = self.password_entry.get()


       if not username or not password:

           messagebox.showerror("Error", "Please enter both username and password")

           return


       # Check if using stored credentials

       if (hasattr(self, 'stored_password_hash') and

               password == "●●●●●●●●" and

               not self.auto_login_attempted):

           self.auto_login_attempted = True

           success, user_data = AuthService.verify_stored_credentials(username, self.stored_password_hash)


           if success:

               if self.remember_var.get():

                   CredentialManager.save_credentials(username, self.stored_password_hash)

               else:

                   CredentialManager.clear_credentials()


               messagebox.showinfo("Success", f"Welcome back, {user_data['full_name']}!")

               self.on_success(user_data)

           else:

               messagebox.showerror("Error", "Invalid username or password")

               self.password_entry.delete(0, 'end')

               self.password_entry.entry.config(fg=COLORS['text'])

               if hasattr(self, 'stored_password_hash'):

                   delattr(self, 'stored_password_hash')

               self.auto_login_attempted = False

       else:

           # Normal login with typed password

           success, user_data = AuthService.login_user(username, password)


           if success:

               if self.remember_var.get():

                   password_hash = AuthService.hash_password(password)

                   CredentialManager.save_credentials(username, password_hash)

               else:

                   CredentialManager.clear_credentials()


               messagebox.showinfo("Success", f"Welcome back, {user_data['full_name']}!")

               self.on_success(user_data)

           else:

               messagebox.showerror("Error", "Invalid username or password")

               self.password_entry.delete(0, 'end')

               self.password_entry.entry.config(fg=COLORS['text'])


   def show_register(self):

       """Show registration page"""

       RegisterPage(self.root, lambda: LoginPage(self.root, self.on_success))



class RegisterPage:

  # Registration page UI


   def __init__(self, root, on_back):

       self.root = root

       self.on_back = on_back


       # Clear window

       for widget in root.winfo_children():

           widget.destroy()


       # Configure root grid (same as login page)

       self.root.grid_columnconfigure(0, weight=1)

       self.root.grid_rowconfigure(0, weight=1)


       # Main background frame

       main_frame = tk.Frame(self.root, bg=COLORS['bg'])

       main_frame.grid(row=0, column=0, sticky="nsew")


       main_frame.grid_columnconfigure(0, weight=1)

       main_frame.grid_rowconfigure(0, weight=1)


       # Register card (same as login card)

       card = tk.Frame(

           main_frame,

           bg=COLORS['white'],

           relief='flat',

           borderwidth=0,

           highlightthickness=0,

       )

       card.grid(row=0, column=0, padx=40, pady=40)


       # Inner content container - COMPACT PADDING

       content = tk.Frame(card, bg=COLORS['white'], padx=50, pady=25)

       content.pack(fill='both', expand=True)


       # Header with logo

       icon_path = resource_path("LOGO.png")


       try:

           img = Image.open(icon_path)

           img = img.resize((60, 60), Image.Resampling.LANCZOS)

           self.reg_logo = ImageTk.PhotoImage(img)

           tk.Label(content, image=self.reg_logo, bg=COLORS['white']).pack(pady=(0, 8))

       except Exception as e:

           print(f"Register Logo Error: {e}")

           tk.Label(content, text="🎓", font=('Segoe UI', 36), bg=COLORS['white']).pack(pady=(0, 8))


       tk.Label(

           content, text="Create Account",

           font=('Segoe UI', 18, 'bold'),

           bg=COLORS['white'], fg=COLORS['primary']

       ).pack()


       tk.Label(

           content, text="Join PUP Grade Tracker",

           font=('Segoe UI', 9),

           bg=COLORS['white'], fg=COLORS['text_light']

       ).pack(pady=(0, 20))


       # Registration form - COMPACT SPACING

       self.fullname_entry = ModernEntry(content, "Full Name")

       self.fullname_entry.pack(fill='x', pady=(0, 10))


       self.student_id_entry = ModernEntry(content, "Student ID")

       self.student_id_entry.pack(fill='x', pady=(0, 10))


       self.course_entry = ModernEntry(content, "Course")

       self.course_entry.pack(fill='x', pady=(0, 10))


       self.username_entry = ModernEntry(content, "Username")

       self.username_entry.pack(fill='x', pady=(0, 10))


       self.password_entry = ModernEntry(content, "Password", show="●")

       self.password_entry.pack(fill='x', pady=(0, 10))


       self.confirm_password_entry = ModernEntry(content, "Confirm Password", show="●")

       self.confirm_password_entry.pack(fill='x', pady=(0, 15))


       # Terms checkbox

       terms_frame = tk.Frame(content, bg=COLORS['white'])

       terms_frame.pack(fill='x', pady=(0, 15))


       self.terms_var = tk.BooleanVar()

       tk.Checkbutton(

           terms_frame, text="I agree to the Terms and Conditions",

           variable=self.terms_var,

           bg=COLORS['white'], font=('Segoe UI', 8),

           activebackground=COLORS['white']

       ).pack(side='left')


       # Create Account button

       ModernButton(

           content, "Create Account",

           command=self.handle_register,

           style='success'

       ).pack(fill='x', ipady=5, pady=(0, 12))


       # Back to login

       back_frame = tk.Frame(content, bg=COLORS['white'])

       back_frame.pack(pady=(0, 8))


       tk.Label(

           back_frame, text="Already have an account?",

           bg=COLORS['white'], fg=COLORS['text_light'],

           font=('Segoe UI', 9)

       ).pack(side='left')


       back_btn = tk.Label(

           back_frame, text="Login here",

           bg=COLORS['white'], fg=COLORS['primary'],

           font=('Segoe UI', 9, 'bold'),

           cursor='hand2'

       )

       back_btn.pack(side='left', padx=(5, 0))

       back_btn.bind('<Button-1>', lambda e: self.on_back())

       back_btn.bind('<Enter>', lambda e: back_btn.config(fg=COLORS['primary_dark']))

       back_btn.bind('<Leave>', lambda e: back_btn.config(fg=COLORS['primary']))


       # Footer

       tk.Label(

           content, text="© 2026 PUP Grade Tracker. All rights reserved.",

           font=('Segoe UI', 7),

           bg=COLORS['white'], fg=COLORS['text_light']

       ).pack(pady=(15, 0))


       # Bind Enter key for form submission

       self.confirm_password_entry.entry.bind('<Return>', lambda e: self.handle_register())


       # Focus first field

       self.fullname_entry.focus()


   def handle_register(self):

      # Handle registration attempt

       full_name = self.fullname_entry.get().strip()

       student_id = self.student_id_entry.get().strip()

       course = self.course_entry.get().strip()

       username = self.username_entry.get().strip()

       password = self.password_entry.get()

       confirm_password = self.confirm_password_entry.get()


       # Validation

       if not all([full_name, student_id, course, username, password, confirm_password]):

           messagebox.showerror("Error", "All fields are required!")

           return


       if len(username) < 3:

           messagebox.showerror("Error", "Username must be at least 3 characters!")

           return


       if len(password) < 6:

           messagebox.showerror("Error", "Password must be at least 6 characters!")

           return


       if password != confirm_password:

           messagebox.showerror("Error", "Passwords do not match!")

           return


       if not self.terms_var.get():

           messagebox.showerror("Error", "Please agree to the Terms and Conditions!")

           return


       if AuthService.check_username_exists(username):

           messagebox.showerror("Error", "Username already exists!")

           return


       # Register user

       success, message = AuthService.register_user(username, password, student_id, course, full_name)


       if success:

           messagebox.showinfo("Success", "Account created successfully! Please login.")

           self.on_back()

       else:

           messagebox.showerror("Error", message)



class LoginSystem:

   # Main login system controller"


   def __init__(self, root, on_login_success):

       self.root = root

       self.on_login_success = on_login_success


       # Setup authentication database

       AuthService.setup_auth_database()

       CredentialManager.setup_credentials_table()


       # Show login page

       LoginPage(root, self.handle_successful_login)


   def handle_successful_login(self, user_data):

       # Handle successful login and transition to main app

       for widget in self.root.winfo_children():

           widget.destroy()


       self.on_login_success(user_data)



def show_login(root, on_login_success, undo_manager=None):

  $ Show login system

   LoginSystem(root, on_login_success)



def resource_path(relative_path):

   # for logo

       base_path = sys._MEIPASS

   except Exception:

       base_path = os.path.abspath(".")


   return os.path.join(base_path, relative_path)



if __name__ == "__main__":

   root = tk.Tk()

   root.title("PUP Grade Tracker - Login")


   # Set proper window size

   root.geometry("900x800")

   root.minsize(700, 800)


   root.bind("<Escape>", lambda event: root.destroy())

   root.configure(bg=COLORS['bg'])



   def on_success(user_data):

       print(f"Login successful: {user_data}")

       messagebox.showinfo("Success", f"Logged in as {user_data['full_name']}")


   show_login(root, on_success)

   root.mainloop()

