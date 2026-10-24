package io.github.jacomartins.canyonui.data

import android.content.Context
import androidx.room.Dao
import androidx.room.Database
import androidx.room.Entity
import androidx.room.Index
import androidx.room.Insert
import androidx.room.PrimaryKey
import androidx.room.Query
import androidx.room.Room
import androidx.room.RoomDatabase
import androidx.room.Transaction
import androidx.room.Update
import androidx.room.migration.Migration
import androidx.sqlite.db.SupportSQLiteDatabase
import kotlinx.coroutines.flow.Flow

/** A home screen page. Pages are shown in [rank] order. */
@Entity(tableName = "pages")
data class PageEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val rank: Int,
)

/**
 * Anything placed on the home screen, the dock, in a folder or in a widget stack ([type]: app,
 * folder, widget or stack). A widget keeps its provider in [packageName]/[className].
 */
@Entity(tableName = "items", indices = [Index("pageId"), Index("parentId")])
data class ItemEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val type: Int = TYPE_APP,
    /** [io.github.jacomartins.canyonui.model.Container]. */
    val container: Int,
    /** Page id for desktop items, 0 for the dock and for apps in folders. */
    val pageId: Long,
    /** Cell, or for an app in a folder: cellX is its rank in the folder. */
    val cellX: Int,
    val cellY: Int,
    val spanX: Int = 1,
    val spanY: Int = 1,
    /** App package, or a widget's provider package. Empty for folders and stacks. */
    val packageName: String,
    val className: String,
    val userSerial: Long,
    /** When the app was found uninstalled; null while installed. */
    val missingSince: Long? = null,
    /** Folder name (folders only). */
    val title: String? = null,
    /** Folder colour as ARGB; 0 = theme default (folders only). */
    val color: Int = 0,
    /** The folder an app is in (container FOLDER) or the stack a widget is in (STACK), else 0. */
    val parentId: Long = 0,
    /** The id the widget host gave this widget (widgets only), else -1. */
    val appWidgetId: Int = -1,
    /**
     * Folders made by "Organize apps by category": the category's key. New apps of the category
     * join it, and it is kept while the mode is on; null for every other item.
     */
    val category: String? = null,
) {
    companion object {
        const val TYPE_APP = 0
        const val TYPE_FOLDER = 1
        const val TYPE_WIDGET = 2
        const val TYPE_STACK = 3
    }
}

@Dao
interface LauncherDao {
    @Query("SELECT * FROM pages ORDER BY rank")
    fun pages(): Flow<List<PageEntity>>

    @Query("SELECT * FROM items")
    fun items(): Flow<List<ItemEntity>>

    @Query("SELECT * FROM pages ORDER BY rank")
    suspend fun pagesNow(): List<PageEntity>

    @Query("SELECT COUNT(*) FROM items")
    suspend fun itemCount(): Int

    @Query("SELECT * FROM items")
    suspend fun itemsNow(): List<ItemEntity>

    @Insert
    suspend fun insertPage(page: PageEntity): Long

    @Insert
    suspend fun insertPages(pages: List<PageEntity>)

    @Update
    suspend fun updatePages(pages: List<PageEntity>)

    @Query("DELETE FROM pages WHERE id = :id")
    suspend fun deletePage(id: Long)

    @Query("DELETE FROM items")
    suspend fun deleteAllItems()

    @Query("DELETE FROM pages")
    suspend fun deleteAllPages()

    @Update
    suspend fun updateItem(item: ItemEntity)

    @Query("SELECT * FROM items WHERE id = :id")
    suspend fun item(id: Long): ItemEntity?

    @Query("SELECT * FROM items WHERE parentId = :folderId ORDER BY cellX, id")
    suspend fun children(folderId: Long): List<ItemEntity>

    @Query("SELECT appWidgetId FROM items WHERE type = 2")
    suspend fun widgetIds(): List<Int>

    /** Puts back a saved layout exactly (same ids), for undo. */
    @Transaction
    suspend fun replaceAll(pages: List<PageEntity>, items: List<ItemEntity>) {
        deleteAllItems()
        deleteAllPages()
        insertPages(pages)
        insertItems(items)
    }

    @Insert
    suspend fun insertItem(item: ItemEntity): Long

    @Insert
    suspend fun insertItems(items: List<ItemEntity>)

    @Update
    suspend fun updateItems(items: List<ItemEntity>)

    @Query("SELECT * FROM items WHERE id IN (:ids)")
    suspend fun itemsById(ids: List<Long>): List<ItemEntity>

    @Query("DELETE FROM items WHERE id IN (:ids)")
    suspend fun deleteItems(ids: List<Long>)

    /** Appends pages until there are at least [count]; returns all page ids in rank order. */
    @Transaction
    suspend fun ensurePages(count: Int): List<Long> {
        val existing = pagesNow()
        var rank = (existing.maxOfOrNull { it.rank } ?: -1) + 1
        val ids = existing.map { it.id }.toMutableList()
        while (ids.size < count) ids += insertPage(PageEntity(rank = rank++))
        return ids
    }
}

@Database(entities = [PageEntity::class, ItemEntity::class], version = 4, exportSchema = true)
abstract class LauncherDatabase : RoomDatabase() {
    abstract fun dao(): LauncherDao

    companion object {
        /** Stage 2: folders (title, colour, parent link). Existing rows keep their values. */
        val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("ALTER TABLE items ADD COLUMN title TEXT")
                db.execSQL("ALTER TABLE items ADD COLUMN color INTEGER NOT NULL DEFAULT 0")
                db.execSQL("ALTER TABLE items ADD COLUMN parentId INTEGER NOT NULL DEFAULT 0")
                db.execSQL("CREATE INDEX IF NOT EXISTS index_items_parentId ON items (parentId)")
            }
        }

        /** Stage 3: widgets (the host's widget id). */
        val MIGRATION_2_3 = object : Migration(2, 3) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("ALTER TABLE items ADD COLUMN appWidgetId INTEGER NOT NULL DEFAULT -1")
            }
        }

        /** Category folders (home screen only mode). */
        val MIGRATION_3_4 = object : Migration(3, 4) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("ALTER TABLE items ADD COLUMN category TEXT")
            }
        }

        /** One database per layout mode, so each mode keeps its own pages and icons. */
        fun create(context: Context, mode: LayoutMode): LauncherDatabase {
            val name = when (mode) {
                LayoutMode.HOME_AND_DRAWER -> "launcher.db" // Name kept from Stage 1 so layouts survive.
                LayoutMode.HOME_ONLY -> "launcher_home_only.db"
            }
            return Room.databaseBuilder(context, LauncherDatabase::class.java, name)
                .addMigrations(MIGRATION_1_2, MIGRATION_2_3, MIGRATION_3_4)
                .build()
        }
    }
}
